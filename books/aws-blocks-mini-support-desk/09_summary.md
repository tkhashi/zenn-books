---
title: "まとめ"
free: true
---

## この章でやること

ここまでで Mini Support Desk が完成しました。この章では、学んだことを振り返り、AWS Blocks をこれから使うときの指針を整理します。

## 作ったもの

Block を組み合わせるだけで、これだけの機能を持つアプリができました。

- ログイン（`AuthCognito`）
- 問い合わせチケットの作成・一覧・詳細・クローズ（`Database`）
- 添付ファイル（`FileBucket`）
- ログ・メトリクス・ダッシュボード（`Logger` / `Metrics` / `Dashboard`）
- 作成通知の非同期化（`AsyncJob`）
- open チケット数の定期集計（`CronJob`）
- メール通知（`EmailClient`）
- 重要チケットのワークフロー（CDK layer の SQS + Step Functions）
- 運用アラート（CDK layer の CloudWatch Alarm + SNS）
- CI/CD 定義（`Pipeline`）
- 回答案生成の RAG（`KnowledgeBase` + `Agent`）

そして、その大半が **AWS アカウントなしのローカルモック**で開発・確認できました。

## AWS Blocks でできること

- **Block を組み合わせてバックエンドを作る** — 各 Block が CDK construct・SDK 連携・ローカルモックをまとめて持つ
- **型安全な API** — `ApiNamespace` のメソッドを、フロントから `import { api } from 'aws-blocks'` で型付きで呼べる
- **同じコードで 3 モード** — local / sandbox / production を、コードを変えずに切り替えられる

## CDK layer に降りるべき場面

Block で表現できないリソース（SQS・Step Functions・SNS・独自の CloudWatch 設定など）が必要になったら CDK layer に降ります。やることは 3 つだけでした。

1. リソースを作る（`index.cdk.ts`）
2. Blocks の Lambda（`blocksStack.handler`）に権限を渡す
3. 環境変数で ARN / URL を渡す

「Block で済むなら Block、足りなければ CDK layer」という順番で考えるとシンプルです。

## local / sandbox / production の違い

| | local（`npm run dev`） | sandbox（`npm run sandbox`） | production（`npm run deploy`） |
|---|---|---|---|
| 用途 | 開発・単体確認 | 実 AWS での検証 | 本番 |
| AWS | 不要 | 必要 | 必要 |
| 課金 | なし | あり | あり |
| Block | モック | 実サービス | 実サービス |

CDK layer のリソースと Bedrock の「本物」は sandbox で確認しました。**sandbox を使ったら、使い終わりに `npm run sandbox:destroy` で必ず削除**するのを忘れないでください。

## アプリエンジニアが気をつけるべき点

- **認証はオプトイン** — `ApiNamespace` のメソッドは既定で公開。保護したいメソッドでは `auth.requireAuth(context)` を呼ぶ。「自分のデータだけ」は `WHERE owner_sub = ...` のようにアプリ側で保証する。
- **非同期・定期処理は冪等に** — `AsyncJob` / `CronJob` は「少なくとも 1 回」実行。同じ処理が複数回走っても壊れない設計にする。
- **ローカルモックと実サービスの差を意識する** — 特に `KnowledgeBase` のローカル検索は TF-IDF で、日本語の精度や検索スコアは実 Bedrock と異なる。`Dashboard` はクラウド専用。
- **後片付け** — sandbox の Aurora など、起動中に課金されるリソースは放置しない。

## 発展課題

本 Book ではスコープ外とした、さらに先のテーマです。Mini Support Desk を題材に挑戦してみてください。

- 管理者画面 / ロール管理（`requireRole` とグループ）
- コメント機能 / 担当者割り当て
- 承認 UI（Step Functions の人手承認）
- AI 回答履歴の永続化（`Agent` の会話永続化、`inferenceOnly` を外す）
- 検索・フィルタリングの作り込み（`KnowledgeBase` のメタデータフィルタ）
- Secrets Manager / KMS、WAF などのセキュリティ強化
- 本格的な CI/CD と production 運用設計（`Pipeline` の本番運用）

## おわりに

AWS Blocks は「アプリエンジニアが、インフラの定型作業に煩わされずにクラウドアプリを作る」ためのフレームワークです。ローカルで素早く作り、必要なところだけ CDK layer に降り、sandbox で実 AWS を確認する——この流れが手に馴染めば、次のアプリはもっと速く形にできるはずです。

お疲れさまでした！
