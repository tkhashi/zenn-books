---
title: "[本編] AWS Blocks で作る Mini Support Desk の全体像"
free: true
---

## この章でやること

この Book では、**AWS Blocks** というフレームワークを使って「Mini Support Desk」という小さなサポートデスクアプリを段階的に作りながら、AWS Blocks の使い方を学びます。

この章では実装はまだ行いません。まず AWS Blocks とは何か、なぜアプリエンジニアにとって便利なのか、そしてこの Book でどこまで作るのかという全体像をつかみます。

:::message
対象バージョン: 本 Book は `@aws-blocks/*` の **v0.1.x** を前提にしています。AWS Blocks はまだ若いフレームワークで、今後 API が変わる可能性があります。手元のバージョンと挙動が違う場合は `node_modules/@aws-blocks/blocks/README.md` と各 Block の README を正典として確認してください。
:::

## 前提

この Book は、次のような**アプリケーションエンジニア**を想定しています。

- TypeScript で Web アプリ（React などの SPA）を書いたことがある
- REST や RPC で API を呼んだことがある
- AWS アカウントを触ったことはあるが、インフラ専門ではない
- CDK は「読めば何となく分かる」程度でよい
- AWS Blocks は初めて

逆に、VPC 設計や IAM の細部、CDK の高度な設計パターンは深追いしません。「アプリを動かすために必要な分だけ」を扱います。

## AWS Blocks とは何か

AWS Blocks は、**バックエンドとフロントエンドを 1 つの TypeScript プロジェクトで書き、ローカルでそのまま動かし、変更なしで AWS にデプロイできる**フレームワークです。

中心にあるのが **Building Block（以下 Block）** という考え方です。Block は「クラウドの 1 機能」を 1 つのモジュールにまとめたもので、それぞれが次の 3 点をセットで持っています。

1. **CDK construct**（AWS にデプロイするためのインフラ定義）
2. **AWS SDK 連携**（実 AWS サービスを呼ぶ実装）
3. **ローカルモック**（AWS アカウントなしで動く擬似実装）

たとえば「データベースが欲しい」と思ったら `Database` という Block を 1 つ作るだけです。ローカルでは WASM 版 PostgreSQL（PGlite）として動き、AWS にデプロイすると Aurora Serverless v2 になります。**コードは同じまま**です。

```typescript
import { Database, sql } from '@aws-blocks/blocks';

const db = new Database(scope, 'main', { migrationsPath: './aws-blocks/migrations' });

// ローカルでも AWS でも、この呼び出し方は変わらない
const tickets = await db.query(sql`SELECT * FROM tickets`);
```

アプリエンジニアにとっての価値はシンプルです。**「IAM ロールを書く」「SDK のクライアントを初期化する」「ローカル用のモックを自作する」といった定型作業を Block が肩代わりしてくれる**ので、アプリのロジックに集中できます。

## Block の考え方

この Book で使う主な Block は次のとおりです。すべて `@aws-blocks/blocks` という 1 つのパッケージからインポートできます。

| Block | 役割 | ローカル | AWS |
|---|---|---|---|
| `ApiNamespace` | 型安全な API（RPC） | dev サーバー | API Gateway + Lambda |
| `AuthCognito` | ログイン認証 | インメモリモック | Cognito User Pool |
| `Database` | リレーショナル DB | PGlite | Aurora Serverless v2 |
| `FileBucket` | ファイル保存 | ローカルファイル | S3 |
| `Logger` / `Metrics` / `Dashboard` | ログ・メトリクス・ダッシュボード | 標準出力 | CloudWatch |
| `AsyncJob` | 非同期ジョブ | インプロセス実行 | SQS + Lambda |
| `CronJob` | 定期実行 | タイマー | EventBridge Scheduler |
| `EmailClient` | メール送信 | コンソール出力 | SES |
| `KnowledgeBase` | RAG の文書検索 | TF-IDF 簡易検索 | Bedrock KnowledgeBase |
| `Agent` | AI エージェント | canned（擬似応答） | Bedrock |
| `Pipeline` | CI/CD | （synth のみ） | CodePipeline |

「Block で足りないものは CDK で足す」ための仕組み（**CDK layer**）もあります。この Book では SQS + Step Functions などを CDK layer で追加します（第5章）。

## ローカルモック・sandbox・production の違い

AWS Blocks には実行モードが 3 つあります。**同じコード**が、モードによって裏側の実装を切り替えます。

```text
ローカル（npm run dev）:
  すべての Block がインメモリ／ローカルファイルのモックで動く
  AWS アカウント不要・課金なし
  この Book の大半はここで完結する

sandbox（npm run sandbox）:
  実 AWS にデプロイして動作確認する検証環境
  AWS アカウントが必要・実費用が発生する
  CDK layer や Bedrock の「本物」を確認する章で使う

production（npm run deploy）:
  本番デプロイ
  この Book では深入りせず、Pipeline 定義までを扱う
```

この Book では、**第5章（CDK layer の sandbox）と第7章（Bedrock の sandbox）以外はすべてローカルモックだけで完結**します。実 AWS を使う章には「費用の目安」と「リソースの削除」を必ず付けています。

## 今回作るアプリの全体構成

作るのは **Mini Support Desk** です。ユーザーが問い合わせチケットを作成し、添付ファイルを付け、必要に応じて AI による回答案を生成できる、最小限のサポートデスクです。

最終的な構成はこうなります。

```text
SPA フロントエンド（React + Vite）
  ↓ 型安全な呼び出し（import { api } from 'aws-blocks'）
ApiNamespace（RPC）
  ↓
Mini Support Desk バックエンド
  ├─ AuthCognito         ... ログイン
  ├─ Database            ... tickets / notification_logs / workflow_logs
  ├─ FileBucket          ... 添付ファイル
  ├─ Logger / Metrics / Dashboard
  ├─ AsyncJob            ... 作成通知の非同期化
  ├─ CronJob             ... open チケット数の定期集計
  ├─ EmailClient         ... メール通知
  ├─ KnowledgeBase       ... 参照ドキュメント検索
  └─ Agent               ... 回答案の生成

CDK layer（第5章で追加）
  ├─ SQS Queue
  ├─ Step Functions
  ├─ SNS Topic
  └─ CloudWatch Alarm

Pipeline（第6章で定義）
```

アプリの作り込み自体は目的ではありません。**各 Block を自然に差し込める題材**として Mini Support Desk を使う、という位置づけです。

## この Book の進め方（本編と任意章）

この Book は **01〜04章を本編**とし、**AWS アカウントなしで** AWS Blocks の基本的な開発体験をつかめる構成にしています。05章以降は任意です。実 AWS・Bedrock・CodePipeline・Ollama などを扱うため、アカウント設定・課金・外部依存が発生します。05章以降を飛ばしても、AWS Blocks の基本は本編だけで学べます。

各章のタイトルに区分を付けています。

| 区分 | 章 | 内容 |
|---|---|---|
| 本編 | 01〜04章 | ローカルモックで完結。AWS 不要 |
| 任意・課金あり | 05章 | CDK layer を sandbox（実 AWS）で確認 |
| 任意 | 06章 | Pipeline を `cdk synth` まで（実デプロイは任意） |
| 任意・課金あり | 07章 | Bedrock RAG を sandbox（実 Bedrock）で確認 |
| おまけ | 08章 | Ollama でローカル LLM |
| おまけ | 09章 | API を RawRoute（生 HTTP）で実装し直す（ローカルのみ） |
| まとめ | 10章 | 振り返りとトラブルシューティング |

:::message
**検証範囲について。** 本 Book のサンプルは、ローカルモックで動く範囲（01〜04章のバックエンドとフロント、およびおまけ 09章の RawRoute 版）を著者環境で動作確認しています。実 AWS・Bedrock・Ollama を使う 05〜08章は、公開時点では**未検証（手順紹介）**です。各章でその旨を明記しています。「著者環境で成功済み」か「手順紹介のみ」かを区別して読み進めてください。
:::

## 検証環境

この Book は次の環境を前提にしています。AWS Blocks は Preview のため、最新バージョンでは API や生成されるテンプレートが変わる可能性があります。再現性のため、サンプルリポジトリの `package-lock.json` に固定された AWS Blocks バージョンを基準にしてください。

- **Node.js**: 22.x（`>=22.0.0`）
- **npm**: 10.x 以上（`>=10.0.0`）
- **AWS Blocks**: `package-lock.json` に固定（検証時点で `@aws-blocks/blocks@0.1.2`）
- **AWS CLI**: v2（05章以降で実 AWS を使う場合）
- **AWS CDK**: `aws-cdk-lib@2.260.0`（`package.json` / lockfile のバージョン）

## 参照

- [AWS Blocks Developer Guide](https://docs.aws.amazon.com/blocks/latest/devguide/what-is-blocks.html)
- [Getting started with AWS Blocks](https://docs.aws.amazon.com/blocks/latest/devguide/getting-started.html)
- [Deploy your application to AWS](https://docs.aws.amazon.com/blocks/latest/devguide/deploy-to-aws.html)
- [AWS Blocks concepts](https://docs.aws.amazon.com/blocks/latest/devguide/concepts.html)
- [AWS Blocks GitHub Repository](https://github.com/aws-devtools-labs/aws-blocks)
- [create-blocks-app README](https://github.com/aws-devtools-labs/aws-blocks/blob/main/packages/create-blocks-app/README.md)

:::message
**フロントエンドコードについて**
本 Book は AWS Blocks（バックエンド）の使い方に集中します。フロントエンド（React）のコードは要点だけ示し、詳細はサンプルリポジトリ [`tkhashi/aws-blocks-mini-support-desk`](https://github.com/tkhashi/aws-blocks-mini-support-desk) を参照してください。バックエンドのコードは本文にすべて掲載するので、写経すれば動きます。
:::

## この章でできたこと

- AWS Blocks が「Block を組み合わせてバックエンドを作る」フレームワークだと理解した
- ローカル / sandbox / production の 3 モードがあり、同じコードで切り替わることを理解した
- これから作る Mini Support Desk の全体像をつかんだ

## 次の章でやること

次の章では、実際にプロジェクトを作成し、`npm run dev` でローカル起動します。AWS Blocks プロジェクトの骨格（`Scope` と `ApiNamespace`）にも触れます。
