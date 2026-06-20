---
title: "非同期・定期処理とメール通知"
free: true
---

## この章でやること

リクエスト・レスポンスの「外」で動く処理を 3 つ足します。

1. **`AsyncJob`** — チケット作成通知を非同期化する
2. **`CronJob`** — open チケット数を定期集計する
3. **`EmailClient`** — メールで通知する

あわせて通知履歴を残す `notification_logs` テーブルも追加します。すべてローカルモックで動きます。

## 前提

- 第3章までのチケット CRUD ができていること

## AsyncJob とは

チケット作成のような操作で「通知も送りたい」とき、API のレスポンスの中で同期的に通知まで処理すると、レスポンスが遅くなります。`AsyncJob` を使うと、重い処理をジョブとして切り出し、API はすぐ返せます。

```text
createTicket
  ↓
tickets に保存
  ↓
AsyncJob.submit  ← ジョブを投入（即座に返る）
  ↓
API は即時レスポンス
  ↓（別途）
ジョブハンドラが通知処理を実行
```

ローカルではインプロセス（同一 Node プロセス内）で実行され、AWS では SQS + Lambda になります。

## 実装

### notification_logs テーブルを追加する

通知の履歴を残すテーブルを、新しいマイグレーションファイルで追加します。

```sql:aws-blocks/migrations/002_create_notification_logs.sql
CREATE TABLE notification_logs (
  id         TEXT PRIMARY KEY,
  ticket_id  TEXT NOT NULL,
  type       TEXT NOT NULL,
  status     TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

ファイルを足すだけで、次回起動時に自動適用されます（`001` は適用済みなのでスキップされます）。

### EmailClient を設定する

先に通知に使う `EmailClient` を作ります。`fromAddress` は送信元アドレスで、AWS では SES で検証済みである必要があります。

```typescript
import { EmailClient } from '@aws-blocks/blocks';

const email = new EmailClient(scope, 'mail', {
  fromAddress: 'support@example.com',
});
```

メール送信は `send()` です。ローカルではコンソールに出力され、`.bb-data` にも記録されます（実際の送信は行われません）。

```typescript
const { messageId } = await email.send({
  to: 'support@example.com',
  subject: '件名',
  body: '本文',
});
```

:::message
`send()` は失敗すると例外を投げます。複数宛先にまとめて送る `sendBatch()` は例外を投げず、結果配列で各件の成否を返します。用途で使い分けます。
:::

### AsyncJob を定義する

ジョブは `handler` を持つ `AsyncJob` として定義します。ハンドラの第2引数 `ctx` には `jobId` などが入ります。

```typescript
import { AsyncJob } from '@aws-blocks/blocks';

const newId = (p = 't') => `${p}_${Date.now().toString(36)}${Math.floor(performance.now()).toString(36)}`;

const ticketCreatedJob = new AsyncJob(scope, 'ticket-created', {
  handler: async (payload: { ticketId: string; title: string }, ctx) => {
    // 通知履歴を記録
    await db.execute(sql`
      INSERT INTO notification_logs (id, ticket_id, type, status)
      VALUES (${newId('n')}, ${payload.ticketId}, 'ticket_created', 'sent')
    `);
    // メール通知
    await email.send({
      to: 'support@example.com',
      subject: `新しい問い合わせ: ${payload.title}`,
      body: `チケット ${payload.ticketId} が作成されました。`,
    });
    log.info('notification sent', { ticketId: payload.ticketId, jobId: ctx.jobId });
  },
});
```

### createTicket からジョブを投入する

第3章の `createTicket` の末尾に、ジョブの投入を 1 行足します。`submit()` は `{ jobId }` を返し、すぐに次へ進みます。

```typescript
  async createTicket(title: string, body: string, priority: TicketPriority = 'normal') {
    const user = await auth.requireAuth(context);
    const id = newId();
    await db.execute(sql`
      INSERT INTO tickets (id, owner_sub, title, body, priority)
      VALUES (${id}, ${user.userSub}, ${title}, ${body}, ${priority})
    `);
    metrics.emit('RequestCreated', 1, { unit: 'Count' });
    const { jobId } = await ticketCreatedJob.submit({ ticketId: id, title }); // ← 追加
    log.info('ticket created', { id, priority, jobId });
    return await db.queryOne<Ticket>(sql`SELECT * FROM tickets WHERE id = ${id}`);
  },
```

#### リトライと冪等性

`AsyncJob` は失敗時に自動でリトライします（既定で最大 3 回、その後はデッドレターキューへ）。リトライがあるということは、**同じジョブが 2 回以上実行されうる**ということです。ハンドラは何度実行されても結果が変わらないよう、**冪等**に書くのが原則です（たとえば「すでに通知済みなら何もしない」など）。AWS Blocks は冪等性そのものは保証しないので、設計はアプリの責任です。

### CronJob で定期集計する

`CronJob` は定期実行の Block です。`schedule` に `rate(...)` か `cron(...)` を指定します。タイムゾーンは IANA 名（`Asia/Tokyo` など）で指定できます。

```typescript
import { CronJob } from '@aws-blocks/blocks';

new CronJob(scope, 'open-ticket-report', {
  schedule: 'rate(1 day)',     // 例: 1 日 1 回
  timezone: 'Asia/Tokyo',
  handler: async (event) => {
    const row = await db.queryOne<{ count: string }>(
      sql`SELECT count(*)::text AS count FROM tickets WHERE status = 'open'`
    );
    log.info('open ticket report', { open: row?.count, at: event.scheduledTime });
    await email.send({
      to: 'support@example.com',
      subject: 'open チケット数レポート',
      body: `現在の open チケット数: ${row?.count}`,
    });
  },
});
```

`CronJob` には `submit()` のような実行時メソッドはありません。**スケジュールに従って自動で `handler` が呼ばれる**だけです。ローカルでは `rate(...)` はタイマー（`setInterval`）で再現されます。手元で素早く確認したいときは `rate(1 minute)` にすると 1 分ごとに走ります。

:::message
定期処理も「少なくとも 1 回」実行のため、重複や、前回が長引いた場合の多重実行が起こりえます。CronJob のハンドラも冪等に書きましょう。
:::

## 動作確認

`npm run dev` を起動し、ログインしてチケットを作成します。`npm run dev` のターミナルに、次のような出力が出れば成功です。

```text
[Email:mail]   ← EmailClient のコンソール出力
{"level":"info","message":"notification sent","ticketId":"t_xxx","jobId":"..."}
```

`CronJob` を `rate(1 minute)` にしておくと、起動時に `[CronJob:open-ticket-report] scheduled: rate(1 minute(s))` と表示され、1 分ごとに `open ticket report` ログが出ます。

`notification_logs` に履歴が入っていることは、API を 1 つ足すか、`.bb-data` のデータで確認できます。

## この章でできたこと

- `AsyncJob` でチケット作成通知を非同期化し、リトライと冪等性の考え方を理解した
- `CronJob` で open チケット数を定期集計した（タイムゾーン指定込み）
- `EmailClient` で通知メールを送った（ローカルはコンソール出力）
- `notification_logs` テーブルを追加した

## 次の章でやること

ここまでは Block だけで完結していました。次の章では Block で足りない AWS リソース（SQS + Step Functions など）を **CDK layer** で追加し、初めて **sandbox** で実 AWS にデプロイして動作確認します。実費用が発生する章なので、費用の目安と削除手順も扱います。
