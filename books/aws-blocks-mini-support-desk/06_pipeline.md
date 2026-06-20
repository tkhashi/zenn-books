---
title: "[任意] Blocks Pipeline を定義して synth する"
free: true
---

## この章でやること

`Pipeline` を使って CI/CD を定義します。本編では `synth`（CloudFormation への合成）までを扱い、実際に AWS 上でパイプラインを動かす手順は章後半でオプションとして紹介します。

## 前提

- 第5章までの実装
- synth までならローカルで完結します（AWS アカウント不要）。実デプロイのオプション部分のみ AWS が必要です。

## Blocks Pipeline とは

`Pipeline` は **CDK Pipelines をラップした CI/CD construct** です。GitHub の特定ブランチへの push をトリガに、`synth`（ビルド + `cdk synth`）→ 自己更新（self-mutating）→ ステージごとのデプロイ、を行う **CodePipeline V2** を作ります。GitHub との接続は **AWS CodeConnections**（OAuth）を使うため、トークン管理は不要です。

```text
GitHub push
  ↓
synth（npm ci → cdk synth）
  ↓
self-mutate（パイプライン定義が変わっていれば自分を更新）
  ↓
staging へデプロイ
  ↓
（手動承認）
  ↓
production へデプロイ
```

:::message
会社で標準の CI/CD（GitHub Actions など）がすでにある場合、`Pipeline` を無理に使う必要はありません。「AWS Blocks には CDK Pipelines ベースの CI/CD も用意されている」という選択肢として理解しておけば十分です。
:::

## 実装

### Pipeline を定義する

パイプライン定義は、専用の CDK アプリファイル（例: `aws-blocks/pipeline.cdk.ts`）に書くのが分かりやすいです。`Pipeline.create()` は非同期ファクトリで、各ステージで `BlocksStack.create()`（これも非同期）を呼べます。

```typescript:aws-blocks/pipeline.cdk.ts
import * as cdk from 'aws-cdk-lib';
import { join, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
import { Pipeline, BlocksStack } from '@aws-blocks/blocks/cdk';

const __dirname = dirname(fileURLToPath(import.meta.url));
const app = new cdk.App();
const pipelineStack = new cdk.Stack(app, 'mini-support-desk-pipeline');

await Pipeline.create(pipelineStack, 'Pipeline', {
  // ① ソース: GitHub リポジトリと CodeConnections の ARN
  source: {
    repo: 'tkhashi/aws-blocks-mini-support-desk',
    connectionArn: process.env.CODECONNECTIONS_ARN!,
  },
  // ② ブランチごとのパイプライン。main は staging → production の順でデプロイ
  branches: [
    {
      branch: 'main',
      stages: [
        { name: 'staging' },
        { name: 'production', requireApproval: true }, // 本番前に手動承認
      ],
    },
  ],
  // ③ 各ステージに何をデプロイするか
  stageFactory: async (stage, stageConfig) => {
    await BlocksStack.create(stage, 'App', {
      backendHandlerPath: join(__dirname, 'index.handler.ts'),
      backendCDKPath: join(__dirname, 'index.ts'),
      stackName: `mini-support-desk-${stageConfig.name}`,
    });
  },
});
```

主なプロパティを整理します（型は `Pipeline` construct の定義そのままです）。

| プロパティ | 役割 |
|---|---|
| `source.repo` | `owner/repo` 形式のリポジトリ |
| `source.connectionArn` | CodeConnections 接続の ARN |
| `branches[].branch` | トリガとなる Git ブランチ |
| `branches[].stages[]` | デプロイ順のステージ。`requireApproval` で手動承認、`bakeTime` で待機 |
| `stageFactory` | 各ステージに配置するスタックを組み立てる関数 |
| `synth` | synth ステップのコマンド（既定 `['npm ci', 'npx cdk synth']`） |

### staging / production と手動承認

上の例では `branches` の `main` に対して `staging` → `production` の 2 ステージを定義しました。`production` に `requireApproval: true` を付けることで、本番デプロイ前に CodePipeline 上で**手動承認**が要求されます。承認するまで production には進みません。

### ローカルで synth する

実デプロイの前に、定義が正しく CloudFormation に変換（合成）できるかをローカルで確認します。これが **synth** です。`Pipeline` は標準の CDK アプリなので、`cdk synth` で合成できます。

```bash
npx cdk synth --app "npx tsx aws-blocks/pipeline.cdk.ts"
```

エラーなく CloudFormation テンプレートが出力されれば、定義は妥当です。**ここまでは AWS アカウント不要**で、課金も発生しません。

:::message
**この章の `cdk synth` は公開時点では未検証**です（コード例と手順の提示）。実行する場合は、上記コマンドで CloudFormation テンプレートが出力されることを確認してください。`cdk synth` までは AWS アカウント不要・課金なしです。
:::

:::message
AWS Blocks のテンプレートには `synth` 専用の npm スクリプトはありません。synth は標準の `cdk synth` を使います。
:::

## （オプション）Pipeline を AWS 上で動かす

:::message alert
ここから先はオプションです。AWS アカウントと GitHub 接続が必要で、実際に AWS リソース（CodePipeline・CodeBuild など）が作成され、**費用が発生**します。
:::

実際にパイプラインを動かすには、次の手順を踏みます。

1. **CodeConnections 接続を作成する**
   AWS コンソールの **Developer Tools → Connections** で GitHub への接続を作成します。作成直後は `PENDING` 状態なので、「保留中の接続を更新」から GitHub App を認可し、対象リポジトリを選びます。`AVAILABLE` になれば使えます。発行された ARN を `CODECONNECTIONS_ARN` 環境変数に入れます。
2. **パイプラインスタックをデプロイする**
   ```bash
   CODECONNECTIONS_ARN=arn:aws:codeconnections:... \
     npx cdk deploy --app "npx tsx aws-blocks/pipeline.cdk.ts"
   ```
3. **ブランチに push する**
   `main` に push すると、パイプラインが起動して synth → staging デプロイへ進みます。
4. **production を承認する**
   staging が成功したら、CodePipeline の画面で production への手動承認を行います。

### 注意事項と課金について

- CodePipeline / CodeBuild / 各ステージのリソースに費用が発生します。
- 試し終わったら、パイプラインスタックと各ステージのスタックを削除してください。
  ```bash
  npx cdk destroy --app "npx tsx aws-blocks/pipeline.cdk.ts"
  ```
- CodeConnections の接続も、不要なら削除します。

## この章でできたこと

- `Pipeline` が CDK Pipelines ベースの CI/CD construct であることを理解した
- staging → production と手動承認を含むパイプラインを定義した
- `cdk synth` でローカル合成する手順を理解した（**公開時点では `cdk synth` 未検証**）
- 実 AWS で動かす手順（CodeConnections・デプロイ・承認）をオプションとして把握した

## 完了条件

この章は公開時点では `cdk synth` 未検証です。実行する場合は、次を確認してください。

- `aws-blocks/pipeline.cdk.ts` を作成できる
- `npx cdk synth --app "npx tsx aws-blocks/pipeline.cdk.ts"` が成功する
- 実デプロイには CodeConnections と AWS 課金が必要だと理解できる

## 次の章でやること

次の章では、いよいよ AI 機能です。`KnowledgeBase` で参照ドキュメントを検索し、`Agent` で回答案を生成する RAG を、ローカルモックから sandbox の実 Bedrock まで通して作ります。
