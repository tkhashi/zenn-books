---
title: "[任意・課金あり] CDK layer で AWS リソースを追加する（SQS + Step Functions / sandbox / Alarm）"
free: true
---

## この章でやること

ここまでは Block だけで完結していました。しかし「Block にない AWS リソースを足したい」場面は必ず出てきます。AWS Blocks ではそれを **CDK layer** で行います。

この章では次を行います。

1. CDK layer の考え方（3 ステップ）を理解する
2. 重要チケット（high）向けに **SQS + Step Functions** のワークフロー**骨格**を追加する
3. 初めて **sandbox** で実 AWS にデプロイして動作確認する手順を理解する
4. **CloudWatch Alarm + SNS** で運用アラートを追加する

:::message alert
**この章は実 AWS を使います。** CDK layer で追加したリソースはローカルモックがないため、動作確認は sandbox（実 AWS デプロイ）で行います。AWS アカウントが必要で、**実費用が発生**します。章末の「費用の目安」と「リソースの削除」を必ず確認してください。

なお、**この章の sandbox 実行（実 AWS）は公開時点では未検証**です。コード例と手順を提示しています。実行する場合は、章末の確認観点に沿って検証し、完了後に `npm run sandbox:destroy` で必ず削除してください。
:::

## 前提

- 第4章までの実装
- AWS アカウントと、CLI で使える認証情報
- **AWS CLI v2 が設定済み**であること
- `aws sts get-caller-identity` でアカウント情報が取得できること
- 初回のみ、対象アカウント・リージョンで **CDK bootstrap 済み**であること

`npm run sandbox` の前に、認証と bootstrap を確認しておきます。

```bash
# 認証確認（アカウント ID などが返ればOK）
aws sts get-caller-identity

# 初回のみ: 対象アカウント・リージョンを bootstrap する
npx cdk bootstrap aws://ACCOUNT_ID/REGION
```

CDK bootstrap は、AWS アカウントとリージョンの組み合わせごとに**初回 1 回だけ**必要です。すでに同じアカウント・リージョンで実行済みの場合は不要です。

## CDK layer とは

CDK layer を使うときの考え方は、たった 3 ステップに整理できます。

1. **リソースを作る** — `aws-blocks/index.cdk.ts` に CDK リソースを定義する
2. **Blocks の Lambda に権限を渡す** — `xxx.grantYYY(blocksStack.handler)` で IAM を付与する
3. **環境変数で ARN や URL を渡す** — `blocksStack.handler.addEnvironment(...)` でバックエンドから参照できるようにする

これ以上の高度な CDK 設計は本 Book では扱いません。「リソースを作って、Blocks の Lambda に権限と接続情報を渡す」——これだけ押さえれば十分です。

`index.cdk.ts` の骨格は次のようになっています（`create-blocks-app` が生成したもの）。`blocksStack` がスタック、**`blocksStack.handler` がバックエンドの Lambda 関数**です。ここに CDK リソースを足していきます。

```typescript:aws-blocks/index.cdk.ts
import * as cdk from 'aws-cdk-lib';
import { RemovalPolicies, Mixins } from 'aws-cdk-lib';
import { BlocksStack, Hosting, SandboxDisableDeletionProtection } from '@aws-blocks/blocks/cdk';
// ...

export const blocksStack = await BlocksStack.create(app, stackName, {
  backendHandlerPath: join(__dirname, 'index.handler.ts'),
  backendCDKPath: join(__dirname, 'index.ts'),
});

// ★ ここから下に CDK layer のリソースを足していく
```

## AsyncJob との使い分け

第4章の `AsyncJob` と何が違うのか、と思うかもしれません。使い分けはこうです。

```text
AsyncJob:
  単発の非同期処理（通知を 1 回送る、など）

SQS + Step Functions:
  複数ステップの非同期ワークフロー
  分岐・可視化・長めの業務処理・外部連携
```

ここでは「high 優先度のチケットは、検証 → トリアージ要フラグ → 通知 という複数段階の処理に流す」というワークフローを Step Functions で組みます。

## 実装

### workflow_logs テーブルを追加する

ワークフローの実行記録を残すテーブルを追加します。

```sql:aws-blocks/migrations/003_create_workflow_logs.sql
CREATE TABLE workflow_logs (
  id            TEXT PRIMARY KEY,
  ticket_id     TEXT NOT NULL,
  execution_arn TEXT,
  status        TEXT NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### SQS Queue と Step Functions を定義する

処理フローはこうです。

```text
high チケット作成 → SQS にメッセージ → Step Functions 起動
  → validate(Pass) → markTriageRequired(Pass) → sendNotification(Pass)
```

:::message
この章の Step Functions は、CDK layer で **SQS・EventBridge Pipes・Step Functions を接続する形**を学ぶための**骨格**です。各ステップは `Pass` 状態（何もせず通過する状態）であり、DB 更新・通知送信・`workflow_logs` への記録といった**実処理は行いません**。実務では各 `Pass` を `sfn-tasks` の `LambdaInvoke` などに置き換えて実処理を割り当てます。本章の目的は「CDK layer によるリソース接続の学習」です。
:::

`index.cdk.ts` の `blocksStack` 作成後に、リソースを定義します。

```typescript:aws-blocks/index.cdk.ts
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';

// ① リソースを作る — SQS Queue
const triageQueue = new sqs.Queue(blocksStack, 'TriageQueue', {
  visibilityTimeout: cdk.Duration.seconds(60),
});

// ① リソースを作る — Step Functions（3 ステップの簡単なワークフロー）
const validate = new sfn.Pass(blocksStack, 'Validate');
const markTriage = new sfn.Pass(blocksStack, 'MarkTriageRequired', {
  result: sfn.Result.fromObject({ status: 'triage_required' }),
});
const notify = new sfn.Pass(blocksStack, 'SendNotification');

const triageWorkflow = new sfn.StateMachine(blocksStack, 'TriageWorkflow', {
  definitionBody: sfn.DefinitionBody.fromChainable(
    validate.next(markTriage).next(notify)
  ),
});
```

:::message
ここでは各ステップを `Pass`（何もせず通過する状態）で表現し、ワークフローの「形」に集中しています。実務では `sfn-tasks` の `LambdaInvoke` などで、各ステップに実処理を割り当てます。本 Book では CDK の深掘りはここまでとします。
:::

### SQS と Step Functions をつなぐ

SQS に届いたメッセージで Step Functions を起動します。コードを書かずにつなげる **EventBridge Pipes** を使います。

```typescript:aws-blocks/index.cdk.ts
import * as pipes from 'aws-cdk-lib/aws-pipes';
import * as iam from 'aws-cdk-lib/aws-iam';

const pipeRole = new iam.Role(blocksStack, 'TriagePipeRole', {
  assumedBy: new iam.ServicePrincipal('pipes.amazonaws.com'),
});
triageQueue.grantConsumeMessages(pipeRole);
triageWorkflow.grantStartExecution(pipeRole);

new pipes.CfnPipe(blocksStack, 'TriagePipe', {
  roleArn: pipeRole.roleArn,
  source: triageQueue.queueArn,
  target: triageWorkflow.stateMachineArn,
  targetParameters: {
    stepFunctionStateMachineParameters: { invocationType: 'FIRE_AND_FORGET' },
  },
});
```

### Blocks の Lambda に権限と接続情報を渡す

②権限付与と③環境変数の注入です。バックエンドが SQS に送信できるようにします。

```typescript:aws-blocks/index.cdk.ts
// ② Blocks の Lambda に SQS 送信権限を渡す
triageQueue.grantSendMessages(blocksStack.handler);

// ③ 環境変数で Queue URL を渡す
blocksStack.handler.addEnvironment('TRIAGE_QUEUE_URL', triageQueue.queueUrl);
```

### Blocks API から SQS に送信する

バックエンド（`aws-blocks/index.ts`）側で、high チケットのときに SQS へ送ります。Lambda 実行環境には AWS SDK が使えます。

```typescript:aws-blocks/index.ts
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';

const sqsClient = new SQSClient({});

// createTicket の中、INSERT の後に追加
if (priority === 'high' && process.env.TRIAGE_QUEUE_URL) {
  await sqsClient.send(new SendMessageCommand({
    QueueUrl: process.env.TRIAGE_QUEUE_URL,
    MessageBody: JSON.stringify({ ticketId: id, title }),
  }));
  await db.execute(sql`
    INSERT INTO workflow_logs (id, ticket_id, status)
    VALUES (${newId('w')}, ${id}, 'queued')
  `);
}
```

`process.env.TRIAGE_QUEUE_URL` はローカルでは未設定なので、この分岐は **ローカルでは自然にスキップ**されます。チケット作成自体はローカルでも従来どおり動きます。

## CloudWatch Alarm と SNS 通知を追加する

第3章の `Dashboard` は「見るための観測」でした。ここでは「異常時に**気づくための通知**」を CDK layer で足します。

`EmailClient` との使い分けはこうです。

```text
EmailClient:        ユーザー向け・業務向けメール
SNS + CloudWatch Alarm: 運用者向けアラート（エラー急増・ワークフロー失敗など）
```

Step Functions の実行失敗を検知して、SNS 経由でメール通知します。

```typescript:aws-blocks/index.cdk.ts
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subs from 'aws-cdk-lib/aws-sns-subscriptions';
import * as cw from 'aws-cdk-lib/aws-cloudwatch';
import * as cwActions from 'aws-cdk-lib/aws-cloudwatch-actions';

// SNS Topic と購読（運用者のメール）
const alertTopic = new sns.Topic(blocksStack, 'OpsAlerts');
alertTopic.addSubscription(new subs.EmailSubscription('ops@example.com'));

// Step Functions の失敗数を監視する Alarm
const failedAlarm = new cw.Alarm(blocksStack, 'TriageWorkflowFailed', {
  metric: triageWorkflow.metricFailed({ period: cdk.Duration.minutes(5) }),
  threshold: 1,
  evaluationPeriods: 1,
  comparisonOperator: cw.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
});
failedAlarm.addAlarmAction(new cwActions.SnsAction(alertTopic));
```

これで「ワークフローが 5 分間に 1 回でも失敗したら、運用者にメールが飛ぶ」状態になります。

## 動作確認

### sandbox にデプロイする

CDK layer で足したリソースはローカルモックがないため、sandbox にデプロイして確認します。

```bash
npm run sandbox
```

初回はリソース作成に数分かかります。完了するとバックエンドが実 AWS にデプロイされ、ローカルのフロントエンドからその sandbox API を叩けるようになります（CORS は自動設定されます）。

確認する観点は次のとおりです。

1. high 優先度のチケットを作成する
2. AWS コンソールの **SQS** でメッセージが流れたことを確認する
3. **Step Functions** で execution が起動・成功したことを確認する
4. （SNS のメール購読は、初回に届く確認メールで承認しておく）

うまくいかないときは、CloudWatch Logs で Lambda の実行ログ、IAM 権限（`grant...` の付け忘れ）、環境変数（`addEnvironment` の付け忘れ）を確認します。

## 費用の目安

sandbox で作られる主なリソースの目安です（東京リージョン・軽い検証利用の概算。正確な料金は必ず[料金ページ](https://aws.amazon.com/jp/pricing/)で確認してください）。

- **SQS / Step Functions / SNS** … いずれも従量課金。検証レベルのリクエスト数ならごくわずか（多くは無料枠内）。
- **Lambda / CloudWatch Logs** … 同上、検証レベルなら少額。
- **Aurora Serverless v2（Database Block）** … 最小 0.5 ACU でも**起動中は時間課金**が発生します。検証が終わったら必ず削除してください。

**使い終わったら放置せず削除する**のが鉄則です。

## リソースの削除

sandbox のリソースは次のコマンドで一括削除できます。

```bash
npm run sandbox:destroy
```

`index.cdk.ts` では sandbox モード時に `RemovalPolicies.of(blocksStack).destroy()` が効いているため、Aurora などの保護も外れてクリーンに削除されます。削除後、AWS コンソールでスタックが消えていることを念のため確認しましょう。

## この章でできたこと

- CDK layer の 3 ステップ（作る・権限・環境変数）を理解した
- SQS + Step Functions で high チケット向けワークフローの**骨格**を追加した（各ステップは `Pass`）
- sandbox にデプロイして実 AWS で確認する手順を理解した（**公開時点では実 AWS 未検証**）
- CloudWatch Alarm + SNS で運用アラートを追加した
- 費用を理解し、`sandbox:destroy` で後片付けする流れを把握した

## 完了条件

この章は公開時点では**実 AWS 未検証**です。実行する場合は、次を確認してください。

- `aws sts get-caller-identity` が成功する
- 対象アカウント・リージョンで CDK bootstrap が済んでいる
- `npm run sandbox` が成功する
- SQS / EventBridge Pipes / Step Functions / SNS / CloudWatch Alarm が作成される
- high チケット作成時に SQS へメッセージが送信される
- EventBridge Pipes 経由で Step Functions execution が起動し、各ステップが `Pass` として成功する
- SNS 購読確認メールを承認できる
- `npm run sandbox:destroy` でリソースを削除できる

## 次の章でやること

次の章では、CI/CD を定義する `Pipeline` を扱います。本編では `synth`（合成）まで、実 AWS でのデプロイはオプションとして紹介します。
