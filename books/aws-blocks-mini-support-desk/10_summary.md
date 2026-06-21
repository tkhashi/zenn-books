---
title: "[まとめ] まとめ"
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

CDK layer のリソースと Bedrock の「本物」は sandbox で確認します（本 Book では公開時点で未検証のため、手順として提示しています）。**sandbox を使ったら、使い終わりに `npm run sandbox:destroy` で必ず削除**するのを忘れないでください。

### 実行モードごとに何が起きるか

AWS Blocks の理解で重要なのは、**同じコードが実行モードによって異なる実装に解決される**点です。最後に整理しておきます。

| 実行モード | 何が起きるか | 主な確認対象 |
|---|---|---|
| local（`npm run dev`） | Blocks がローカル実装（モック）で動く | API 配線・型安全・認証フロー・ローカル保存 |
| CDK synthesis（`cdk synth`） | Blocks と CDK layer が CloudFormation に変換される | 生成される AWS リソース・IAM・環境変数 |
| sandbox（`npm run sandbox`） | 実 AWS に一時デプロイされる | IAM・S3・Aurora・SQS・Step Functions・Bedrock など実サービス差分 |
| production（`npm run deploy`） | CloudFormation stack として本番向けにデプロイされる | 長期運用・削除ポリシー・監視・権限設計 |

AWS Blocks のキャッチアップでは、まず **local** で開発体験をつかみ、次に **sandbox** で「ローカルモックでは確認できない差分」を確認する、という順番が効率的です。これが AWS Blocks の中核である **conditional exports / local-first** の思想につながっています。

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

## トラブルシューティング

詰まったときの切り分け表です。local / sandbox / Bedrock / Ollama のどこで起きているかを意識すると原因に近づけます。

| 症状 | 原因候補 | 対処 |
|---|---|---|
| `npm run dev` が起動しない | Node.js が 20 系以下 | Node.js 22+ / npm 10+ に更新する |
| `npm install` で依存解決に失敗する | lockfile と Node.js / npm の組み合わせ不一致 | Node.js 22+ / npm 10+ で `npm ci` または `npm install` をやり直す |
| フロントから API が呼べない | dev server / client server の片方が落ちている | `npm run dev:server` と `npm run dev:client` を個別に起動してログを見る |
| サインアップ後に確認コードが分からない | ローカルではメール送信されない | `npm run dev` のターミナルに出る確認コードを見る（`[auth] ... の確認コード: ...`） |
| ローカル状態をリセットしたい | `.bb-data` にデータが残っている | `rm -rf .bb-data` 後に `npm run dev` を再起動する |
| `npm run sandbox` が失敗する | AWS 認証未設定 | `aws sts get-caller-identity` を実行して確認する |
| `npm run sandbox` が CDK bootstrap 関連で失敗する | CDK bootstrap 未実行 | `npx cdk bootstrap aws://ACCOUNT_ID/REGION` を実行する |
| high チケットで SQS に送信されない | `TRIAGE_QUEUE_URL` 未設定、またはローカル実行 | sandbox 環境で実行し、Lambda 環境変数を確認する |
| Step Functions が起動しない | EventBridge Pipes の IAM 権限・接続設定不足 | Pipe の状態・IAM Role・CloudWatch Logs を確認する |
| SNS メールが届かない | Subscription 未承認 | 確認メールから subscription を承認する |
| Bedrock が動かない | モデルアクセス未有効化 | 対象リージョンで Bedrock の Model access を有効化する |
| KnowledgeBase の検索結果が local と sandbox で違う | ローカルは TF-IDF、sandbox は Bedrock の意味検索 | 差分は仕様。日本語はローカルでヒットしにくい（英単語や文書中の語句で確認） |
| Ollama ではなく canned が返る | Ollama 未起動・モデル未 pull・endpoint 不一致 | `ollama serve` / `ollama pull llama3.1:8b` / endpoint を確認する |

## おわりに

AWS Blocks は「アプリエンジニアが、インフラの定型作業に煩わされずにクラウドアプリを作る」ためのフレームワークです。ローカルで素早く作り、必要なところだけ CDK layer に降り、sandbox で実 AWS を確認する——この流れが手に馴染めば、次のアプリはもっと速く形にできるはずです。

お疲れさまでした！
