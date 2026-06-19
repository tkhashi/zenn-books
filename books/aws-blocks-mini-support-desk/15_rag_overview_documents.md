---
title: "Bedrock RAG の概要と参照ドキュメントを用意する"
free: true
---

## この章でやること

## 前提

## RAG の全体像

```text
ticket.body
  ↓
KnowledgeBase で関連ドキュメント検索
  ↓
Agent が回答案を生成
  ↓
画面に表示
```

## ローカルと sandbox の違い

```text
ローカル（この章〜Ch.17）:
  KnowledgeBase の簡易検索
  Agent の canned provider または Ollama

sandbox（Ch.18）:
  実 Bedrock
  実 KnowledgeBase
  実 embedding / retrieval
```

## 参照ドキュメントを用意する

`docs/knowledge-base/` に以下を作成する。

- `support-policy.md` — サポート対応ポリシー
- `account-request-guide.md` — アカウント発行依頼ガイド
- `incident-response-guide.md` — 障害報告ガイド
- `faq.md` — FAQ

## 実装

### support-policy.md

### account-request-guide.md

### incident-response-guide.md

### faq.md

## この章でできたこと

## 次の章でやること
