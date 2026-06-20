# Zenn Book 仕様書：アプリエンジニアのための AWS Blocks ハンズオン

> レビューフィードバック反映済み確定版

> **【2026-06 更新メモ：実 API・章統合の反映】**
> 本文執筆にあたり、実在する `@aws-blocks/*` フレームワーク（npm scope、v0.1.x）を `create-blocks-app` でスカフォールドして API を検証した。その結果、本仕様書の一部記述は実 API と差異があったため、**最新の正典は `books/aws-blocks-mini-support-desk/` の各章本文**とする。主な確定事項：
> - パッケージは `@aws-blocks/blocks`（アンブレラ）から `Scope` / `ApiNamespace` / 各 Block をインポート。フロントは `aws-blocks` エイリアス。
> - 構成は `new Scope('mini-support-desk')` → `new ApiNamespace(scope, 'api', (context) => ({ ... }))`。API は JSON-RPC（REST ではない）。
> - `Database` は PostgreSQL（ローカル PGlite / 本番 Aurora）。テーブルは番号付き `.sql` マイグレーション。`.bb-data` はローカルモックの永続先。
> - 認証はメソッド単位オプトイン（`auth.requireAuth(context)`）。`AuthCognito` はサインアップ後に確認コードが必要（ローカルは `codeDelivery` フックで出力）。
> - CDK layer は `index.cdk.ts` で `blocksStack.handler` に権限・env を付与。`Pipeline` は `@aws-blocks/core/cdk`（アンブレラ経由）の construct。
> - `Agent` はローカル既定で canned provider、Ollama は `provider: 'openai-api'`。`KnowledgeBase` のローカル検索は TF-IDF（日本語の精度は実 Bedrock と異なる）。
>
> **章構成は 9 章に統合した**（旧 3-6 / 7-9 / 10-12 / 13-14 / 15-18 をそれぞれ 1 章に集約）。新構成は `config.yaml` を参照：
> `01_introduction, 02_create_project, 03_core_blocks, 04_background_and_notify, 05_cdk_layer, 06_pipeline, 07_bedrock_rag, 08_ollama_local, 09_summary`。
>
> 以下の「7. Book 全体構成」以降の旧 20 章構成は、設計意図の記録として残す。

## 1. Book の目的

本 Book は、AWS Blocks を初めて使う TypeScript アプリケーションエンジニア向けに、ローカル開発から sandbox デプロイ、CDK layer による拡張、Bedrock RAG までを段階的に体験できるハンズオンとして構成する。

主目的は、サンプルアプリ自体を作り込むことではなく、AWS Blocks の以下の要素を実装を通して理解することである。

* Blocks の基本概念
* ローカルモックによる開発体験
* Cognito 認証
* Database Block によるデータ永続化
* FileBucket によるストレージ利用
* Logger / Metrics / Dashboard による観測
* AsyncJob / CronJob によるバックグラウンド処理
* EmailClient による通知
* Pipeline construct による CI/CD 定義
* CDK layer による AWS リソース追加
* sandbox による実 AWS サービスでの動作確認
* Bedrock Agent / KnowledgeBase による RAG
* Ollama を使ったローカル LLM テスト

## 2. 対象読者

### 想定読者

* TypeScript で Web アプリケーションを開発できる
* React などの SPA 開発経験がある
* REST API の設計・利用経験がある
* AWS CDK は少し触ったことがある、または読めば理解できる
* インフラ専門ではないアプリケーションエンジニア
* AWS Blocks は未経験

### 前提知識

読者は以下をある程度理解している前提とする。

* TypeScript
* Node.js / npm
* SPA の基本
* API 呼び出しの基本
* AWS アカウントの基本的な使い方
* Cognito や S3 などの概要
* CDK の基本的な概念

### 深く説明しすぎないもの

以下は本 Book の本質ではないため、必要最小限の説明に留める。

* React の基礎
* TypeScript の基礎
* REST API とは何か
* AWS IAM の詳細
* VPC / ネットワーク設計
* CDK の詳細な設計パターン
* Bedrock のモデル選定や RAG 精度改善の高度な話題

## 3. サンプルアプリ概要

### アプリ名

Mini Support Desk

### アプリの位置づけ

Mini Support Desk は、ユーザーが問い合わせチケットを作成し、添付ファイルをアップロードし、必要に応じて AI による回答案を生成できる小規模なサポートデスクアプリである。

ただし、本アプリの目的は完成度の高いサポートシステムを作ることではない。AWS Blocks の各機能を自然に差し込める、最小限の題材として設計する。

### フロントエンドスタック

* React + Vite（AWS Blocks `create` コマンドが生成するデフォルト）
* 各章のフロントエンドコードはサンプルリポジトリを参照してコピーする方式とする
* サンプルリポジトリ: `https://github.com/tkhashi/aws-blocks-mini-support-desk`
* 本 Book 本文ではフロントエンドコードの詳細説明は行わない

### アプリの主要機能

最終的に以下を実装する。

* Cognito ログイン
* チケット作成
* チケット一覧
* チケット詳細
* 添付ファイルアップロード
* ログ出力
* カスタムメトリクス
* ダッシュボード
* チケット作成通知の非同期化
* open ticket 数の定期集計
* SES によるメール通知
* Pipeline 定義と synth
* CDK layer による SQS / Step Functions 追加
* sandbox での実 AWS 動作確認
* CloudWatch Alarm / SNS 通知
* Bedrock KnowledgeBase / Agent による回答案生成
* Ollama を使ったローカル LLM 動作確認

## 4. アプリのスコープ

### 実装する画面

本 Book で扱う画面は最小限とする。

1. ログイン画面
2. チケット一覧画面
3. チケット作成画面
4. チケット詳細画面
5. AI 回答案表示エリア

### 実装しない画面・機能

以下は本 Book の本筋から外れるため、実装対象外とする。

* 管理者専用画面
* 複雑なロール管理
* 担当者割り当て
* コメント機能
* 承認・却下 UI
* 添付ファイル専用管理画面
* AI 回答履歴の永続化
* ワークフロー状態の詳細画面
* 複雑な検索・フィルタリング
* ページネーションの作り込み
* 本番レベルの UI デザイン

必要に応じて、最終章で発展課題として触れる。

## 5. データモデル

Database Block を使う。KVStore は使わない。

### tickets

最初に作成する中心テーブル。

```text
tickets
  - id
  - owner_sub
  - title
  - body
  - status
  - priority
  - attachment_key
  - created_at
  - updated_at
```

### notification_logs

AsyncJob / EmailClient の章で追加する。

```text
notification_logs
  - id
  - ticket_id
  - type
  - status
  - created_at
```

### workflow_logs

CDK layer / Step Functions の章で追加する。

```text
workflow_logs
  - id
  - ticket_id
  - execution_arn
  - status
  - created_at
```

### status の例

```ts
type TicketStatus =
  | 'open'
  | 'triage_required'
  | 'in_progress'
  | 'closed';
```

### priority の例

```ts
type TicketPriority =
  | 'normal'
  | 'high';
```

## 6. 技術構成

### Blocks で扱うもの

```text
- ApiNamespace
- AuthCognito
- Database
- FileBucket
- Logger
- Metrics
- Dashboard
- AsyncJob
- CronJob
- EmailClient
- Pipeline
- Agent
- KnowledgeBase
```

### CDK layer で扱うもの

```text
- SQS Queue
- Step Functions StateMachine
- SNS Topic
- CloudWatch Alarm
- CloudWatch Logs MetricFilter
```

### sandbox で確認するもの

```text
- CDK layer で追加した SQS
- Step Functions の起動
- CloudWatch Alarm / SNS
- Bedrock KnowledgeBase / Agent
- SES の実送信確認
```

### ローカルで確認するもの

```text
- Cognito 認証フローの開発確認
- Database のローカル動作
- FileBucket のローカル保存
- Logger / Metrics の組み込み
- AsyncJob のローカル実行
- CronJob のローカル実行
- EmailClient の呼び出し確認
- KnowledgeBase のローカルモック
- Agent の canned provider または Ollama 実行
- Pipeline の synth
```

## 7. Book 全体構成（確定版）

> レビュー反映: Ch.12+13 統合、Ch.16+17 統合、Pipeline を Part 4 に移動

### Part 1: Blocks だけで Mini Support Desk を作る

#### Ch.01. AWS Blocks で作る Mini Support Desk の全体像
#### Ch.02. プロジェクトを作成してローカル起動する
#### Ch.03. Cognito 認証を追加する
#### Ch.04. Database で Ticket を保存する
#### Ch.05. FileBucket で添付ファイルを扱う
#### Ch.06. Logger / Metrics / Dashboard で観測する

### Part 2: Blocks のバックグラウンド処理と通知を追加する

#### Ch.07. AsyncJob で Ticket 作成通知を非同期化する
#### Ch.08. CronJob で Open Ticket 数を定期集計する
#### Ch.09. EmailClient で SES 通知を送る

### Part 3: CDK layer で AWS リソースを追加する

> Ch.12（CDK layer 概要）は Ch.10 の冒頭セクションに統合

#### Ch.10. CDK layer と SQS + Step Functions で重要チケット処理を追加する
#### Ch.11. sandbox で実 AWS サービスとして動かす（費用目安・削除手順あり）
#### Ch.12. CloudWatch Alarm と SNS 通知を追加する

### Part 4: Pipeline を定義する

> レビュー反映: CDK layer の後に移動

#### Ch.13. Blocks Pipeline を定義して synth する
#### Ch.14. [オプション] Pipeline を AWS 上で動かす

### Part 5: Bedrock RAG を追加する

> Ch.16+17（RAG概要 + ドキュメント準備）を1章に統合

#### Ch.15. Bedrock RAG の概要と参照ドキュメントを用意する
#### Ch.16. KnowledgeBase をローカルモックで動かす
#### Ch.17. Agent で回答案を生成する
#### Ch.18. sandbox で Bedrock を実行する（費用目安・削除手順あり）
#### Ch.19. [おまけ] Ollama でローカル LLM を使う

### Part 6: まとめ

#### Ch.20. まとめ

## 8. Zenn Book ファイル構成

```text
books/
  aws-blocks-mini-support-desk/
    config.yaml
    01_introduction.md
    02_create_project.md
    03_auth_cognito.md
    04_database_ticket.md
    05_filebucket_attachment.md
    06_observability.md
    07_async_job.md
    08_cron_job.md
    09_email_client.md
    10_sqs_stepfunctions.md
    11_sandbox_cdk.md
    12_cloudwatch_alarm_sns.md
    13_pipeline_synth.md
    14_pipeline_deploy_optional.md
    15_rag_overview_documents.md
    16_knowledge_base_local.md
    17_agent_answer.md
    18_bedrock_sandbox.md
    19_ollama_local.md
    20_summary.md
```

## 9. config.yaml

```yaml
title: "アプリエンジニアのためのAWS Blocksハンズオン"
summary: "AWS Blocksを使って、Cognito認証、Database、FileBucket、Observability、AsyncJob、CronJob、EmailClient、Pipeline、CDK layer、SQS、Step Functions、Bedrock RAGを段階的に学ぶTypeScriptハンズオンです。"
topics: ["aws", "typescript", "cdk", "serverless", "bedrock"]
published: false
price: 0
toc_depth: 2
chapters:
  - 01_introduction
  - 02_create_project
  - 03_auth_cognito
  - 04_database_ticket
  - 05_filebucket_attachment
  - 06_observability
  - 07_async_job
  - 08_cron_job
  - 09_email_client
  - 10_sqs_stepfunctions
  - 11_sandbox_cdk
  - 12_cloudwatch_alarm_sns
  - 13_pipeline_synth
  - 14_pipeline_deploy_optional
  - 15_rag_overview_documents
  - 16_knowledge_base_local
  - 17_agent_answer
  - 18_bedrock_sandbox
  - 19_ollama_local
  - 20_summary
```

## 10. 各章の執筆フォーマット

各章は以下の構成に揃える。

```md
# 章タイトル

## この章でやること

## 前提

## 実装

## 動作確認

## この章でできたこと

## 次の章でやること
```

必要に応じて、以下を追加する。

```md
## 補足

## よくあるエラー

## 実務ではどうするか
```

sandbox を使う章（Ch.11, Ch.18）はさらに以下を追加する。

```md
## 費用の目安

## リソースの削除
```

## 11. 執筆方針

### 説明のトーン

* アプリエンジニア向けに説明する
* インフラ用語を出す場合は、具体的なアプリ上の意味に置き換える
* CDK は怖く見せない
* 「この章ではここまでやればよい」を明確にする
* 実務での注意点は補足として入れる

### コードの方針

* コードは最小限にする
* アプリの作り込みより Blocks の利用方法を優先する
* 1章で扱う新概念を増やしすぎない
* 前章のコードとの差分が分かるようにする
* 型定義を明示する
* `shared/` に型・定数を集約する

### フロントエンドコードの扱い

* フロントエンドコードは本文で詳細説明しない
* 各章でサンプルリポジトリ（`https://github.com/tkhashi/aws-blocks-mini-support-desk`）の該当ファイルを参照させる
* コピーして使える形にしておく

### UI の方針

* UI はシンプルにする
* 見た目の作り込みはしない
* CSS やコンポーネント設計に深入りしない
* フォーム、一覧、詳細、ボタン程度で十分とする

### CDK layer の方針

CDK layer の説明は以下の3点に集約する。

```text
1. リソースを作る
2. Blocks の Lambda に権限を渡す
3. 環境変数で ARN や URL を渡す
```

これ以上の高度な CDK 設計は扱わない。

## 12. ローカル / sandbox / optional の扱い

### ローカルで完結する章

```text
01 - 10
13
15 - 17
19
```

### sandbox を使う章

```text
11（費用目安・削除手順 必須）
12
18（費用目安・削除手順 必須）
```

### optional 扱いの章

```text
14（Pipeline 実 AWS デプロイ）
19（Ollama）
```

## 13. 重要な注意事項

### Pipeline

Pipeline は完全なローカルモック対象ではない。
本編では synth までを扱い、実際に pipeline を AWS 上で動かす手順は optional として紹介する。

### CloudWatch

CloudWatch は2段階で扱う。

```text
前半（Ch.06）:
  Blocks の Logger / Metrics / Dashboard

後半（Ch.12）:
  CDK layer の CloudWatch Alarm / MetricFilter / SNS
```

### SQS と AsyncJob

SQS は AsyncJob と単純に重複するものとして扱わない。

```text
AsyncJob（Ch.07）:
  単発の非同期処理

SQS + Step Functions（Ch.10）:
  複数ステップの非同期ワークフロー
  外部連携
  可視化
  分岐
  長めの業務処理
```

### Bedrock

ローカルと sandbox の違いを明確にする。

```text
ローカル（Ch.16-17）:
  KnowledgeBase の簡易検索
  Agent の canned provider または Ollama

sandbox（Ch.18）:
  実 Bedrock
  実 KnowledgeBase
  実 embedding / retrieval
```

### SES

SES は実送信時に以下が必要になる。

```text
- 送信元メールアドレスまたはドメイン検証
- sandbox 制限の理解
- production での送信制限解除
```

## 14. 完成時の構成図

```text
SPA Frontend（React + Vite）
  ↓ type-safe call
ApiNamespace
  ↓
Mini Support Desk Backend
  ├─ AuthCognito
  ├─ Database
  │   ├─ tickets
  │   ├─ notification_logs
  │   └─ workflow_logs
  ├─ FileBucket
  ├─ Logger
  ├─ Metrics
  ├─ Dashboard
  ├─ AsyncJob
  ├─ CronJob
  ├─ EmailClient
  ├─ Agent
  └─ KnowledgeBase

CDK layer
  ├─ SQS Queue
  ├─ Step Functions StateMachine
  ├─ SNS Topic
  ├─ CloudWatch Alarm
  └─ CloudWatch Logs MetricFilter

Pipeline
  └─ staging / production deploy definition
```

## 15. 成功条件

この Book を完了した読者は、以下を説明・実装できる状態を目指す。

* AWS Blocks の基本構成を説明できる
* Blocks だけで小さな Web アプリを作れる
* ローカルモックと sandbox の違いを説明できる
* Database / FileBucket / AuthCognito を使える
* AsyncJob / CronJob / EmailClient を使える
* Blocks Pipeline の位置づけを説明できる
* CDK layer で AWS リソースを追加できる
* SQS + Step Functions を使った非同期ワークフローのイメージを持てる
* CloudWatch Alarm / SNS による運用通知を追加できる
* Bedrock RAG を Blocks で構築する流れを理解できる
* Ollama を使ってローカル LLM テストを試せる
