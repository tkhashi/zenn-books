---
title: "CDK layer と SQS + Step Functions で重要チケット処理を追加する"
free: true
---

## この章でやること

## 前提

## CDK layer とは

CDK layer を使うときの考え方は3ステップに整理できる。

1. **リソースを作る** — `aws-blocks/index.cdk.ts` に CDK リソースを定義する
2. **Blocks の Lambda に権限を渡す** — IAM ポリシーを追加する
3. **環境変数で ARN や URL を渡す** — Lambda から参照できるようにする

これ以上の高度な CDK 設計は本 Book では扱わない。

## AsyncJob との使い分け

```text
AsyncJob:
  単発の非同期処理

SQS + Step Functions:
  複数ステップの非同期ワークフロー
  外部連携
  可視化
  分岐
  長めの業務処理
```

## 処理フロー

```text
High priority ticket 作成
  ↓
tickets に保存
  ↓
SQS にメッセージ送信
  ↓
Step Functions 起動
  ↓
validate ticket
  ↓
mark as triage_required
  ↓
send notification
  ↓
workflow_logs に記録
```

## 実装

### SQS Queue の定義

### Step Functions StateMachine の定義

### Blocks API から SQS 送信

### SQS と Step Functions のつなぎ方

### workflow_logs テーブルの追加

## 動作確認

## この章でできたこと

## 次の章でやること
