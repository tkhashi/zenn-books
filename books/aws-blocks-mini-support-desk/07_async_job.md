---
title: "AsyncJob で Ticket 作成通知を非同期化する"
free: true
---

## この章でやること

## 前提

## AsyncJob とは

```text
createTicket
  ↓
tickets に保存
  ↓
AsyncJob submit
  ↓
API は即時レスポンス
  ↓
Job handler が通知処理を実行
```

## 実装

### AsyncJob の定義

### ticketCreatedJob の実装

### jobId の管理

### notification_logs テーブルの追加

### 失敗時のリトライ

### 冪等性の考え方

## 動作確認

## この章でできたこと

## 次の章でやること
