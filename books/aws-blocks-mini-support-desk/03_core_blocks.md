---
title: "[本編] 認証・データ・ファイル・観測 — Blocks 基礎4種"
free: true
---

## この章でやること

Mini Support Desk の土台を一気に作ります。この章で扱う Block は 4 種類です。

1. **`AuthCognito`** — ログイン認証を付ける
2. **`Database`** — 問い合わせチケットを保存する
3. **`FileBucket`** — 添付ファイルを扱う
4. **`Logger` / `Metrics` / `Dashboard`** — ログ・メトリクス・ダッシュボードで観測する

すべてローカルモックで動かします。**AWS アカウントは不要**です。

## 前提

- 第2章のプロジェクトが `npm run dev` で起動できること

## 実装

この章のコードはすべて `aws-blocks/index.ts` に書いていきます。完成形は章末にまとめます。まずは Block を 1 つずつ足していきましょう。

### 共通の型を用意する

最初に、チケットの型を定義しておきます。`status` と `priority` はユニオン型にしておくと、後の章まで一貫して型チェックが効きます。

```typescript
type TicketStatus = 'open' | 'triage_required' | 'in_progress' | 'closed';
type TicketPriority = 'normal' | 'high';

type Ticket = {
  id: string;
  owner_sub: string;
  title: string;
  body: string;
  status: TicketStatus;
  priority: TicketPriority;
  attachment_key: string | null;
  created_at: string;
  updated_at: string;
};
```

### AuthCognito — ログインを付ける

`AuthCognito` は Amazon Cognito User Pool による認証を提供する Block です。ローカルではインメモリのモックとして動き、サインアップ・確認コード・サインインの一連のフローをそのまま体験できます。

```typescript
import { Scope, AuthCognito } from '@aws-blocks/blocks';

const scope = new Scope('mini-support-desk');

const auth = new AuthCognito(scope, 'auth', {
  passwordPolicy: { minLength: 8 },
  signInWith: 'email',
  // ローカルモック専用: 確認コードをターミナルに出力する
  codeDelivery: async (username, code) => {
    console.log(`[auth] ${username} の確認コード: ${code}`);
  },
});

// サインアップ／サインインの状態遷移をフロントに公開する
export const authApi = auth.createApi();
```

ポイントは次のとおりです。

- **`signInWith: 'email'`** … メールアドレスをユーザー名として使います。
- **確認コードフロー** … Cognito はサインアップ後に確認コードでの本人確認を求めます。これはモックでも同じです。本物の Cognito ではコードはメールで届きますが、ローカルにはメール基盤がありません。そこで `codeDelivery` フック（**モック専用オプション**）を使い、コードをターミナルに出力させています。これで開発中もフローを試せます。
- **`auth.createApi()`** … サインアップ／確認／サインイン／パスワードリセットといった一連の状態遷移を 1 つの API として公開します。フロントの `<Authenticator>` UI はこれを使います。クライアント側の名前空間は `export` した変数名（`authApi`）になります。

:::message
`codeDelivery` はローカルモック用の便利機能です。sandbox / 本番では Cognito（実際にはメール）がコードを配信します。
:::

#### API を認証で保護する

`ApiNamespace` の各メソッドは、**デフォルトでは誰でも呼べる公開エンドポイント**です。認証はメソッドごとに「オプトイン」します。ハンドラの先頭で `auth.requireAuth(context)` を呼ぶと、未ログインのリクエストは `401` で弾かれます。

```typescript
export const api = new ApiNamespace(scope, 'api', (context) => ({
  async whoami() {
    const user = await auth.requireAuth(context); // 未ログインなら 401
    return { sub: user.userSub, email: user.username };
  },
}));
```

`requireAuth` が返すユーザーオブジェクトには、Cognito の一意な ID である `userSub`、ユーザー名（ここではメール）`username`、所属グループ `groups` などが入っています。チケットの所有者を表す `owner_sub` には、この `userSub` を使います。

### Database — チケットを保存する

`Database` は PostgreSQL の Block です。ローカルでは WASM 版 PostgreSQL（PGlite）として `.bb-data` に永続化され、AWS では Aurora Serverless v2 になります。

#### マイグレーションを書く

テーブルは**番号付きの `.sql` ファイル**で定義します。`aws-blocks/migrations/` ディレクトリを作り、次のファイルを置きます。

```sql:aws-blocks/migrations/001_create_tickets.sql
CREATE TABLE tickets (
  id             TEXT PRIMARY KEY,
  owner_sub      TEXT NOT NULL,
  title          TEXT NOT NULL,
  body           TEXT NOT NULL,
  status         TEXT NOT NULL DEFAULT 'open',
  priority       TEXT NOT NULL DEFAULT 'normal',
  attachment_key TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX tickets_owner_created_idx ON tickets (owner_sub, created_at DESC);
```

マイグレーションは**自動で適用**されます。ローカルでは初回クエリ時に、AWS では `cdk deploy` 時に実行されます。適用済みのファイルは `_migrations` テーブルで管理され、各ファイルは一度だけ走ります。

#### Database を定義してクエリする

```typescript
import { Database, sql } from '@aws-blocks/blocks';

const db = new Database(scope, 'main', {
  migrationsPath: './aws-blocks/migrations',
});
```

クエリは `sql` タグ付きテンプレートで書きます。`${}` で埋め込んだ値は自動的にパラメータ化されるので、SQL インジェクションの心配はありません。

```typescript
// 1 件取得
await db.queryOne<Ticket>(sql`SELECT * FROM tickets WHERE id = ${id}`);
// 複数取得
await db.query<Ticket>(sql`SELECT * FROM tickets WHERE owner_sub = ${owner}`);
// 更新・挿入（影響行数が返る）
await db.execute(sql`UPDATE tickets SET status = 'closed' WHERE id = ${id}`);
```

これでチケットの CRUD を API にまとめられます。

```typescript
const newId = () => `t_${Date.now().toString(36)}${Math.floor(performance.now()).toString(36)}`;

export const api = new ApiNamespace(scope, 'api', (context) => ({
  async createTicket(title: string, body: string, priority: TicketPriority = 'normal') {
    const user = await auth.requireAuth(context);
    const id = newId();
    await db.execute(sql`
      INSERT INTO tickets (id, owner_sub, title, body, priority)
      VALUES (${id}, ${user.userSub}, ${title}, ${body}, ${priority})
    `);
    return await db.queryOne<Ticket>(sql`SELECT * FROM tickets WHERE id = ${id}`);
  },

  async listTickets() {
    const user = await auth.requireAuth(context);
    return await db.query<Ticket>(
      sql`SELECT * FROM tickets WHERE owner_sub = ${user.userSub} ORDER BY created_at DESC`
    );
  },

  async getTicket(id: string) {
    const user = await auth.requireAuth(context);
    return await db.queryOne<Ticket>(
      sql`SELECT * FROM tickets WHERE id = ${id} AND owner_sub = ${user.userSub}`
    );
  },

  async closeTicket(id: string) {
    const user = await auth.requireAuth(context);
    await db.execute(
      sql`UPDATE tickets SET status = 'closed', updated_at = now()
          WHERE id = ${id} AND owner_sub = ${user.userSub}`
    );
    return { ok: true };
  },
}));
```

`WHERE owner_sub = ${user.userSub}` を必ず付けることで、**他人のチケットは読めない／触れない**ようにしています。認証はログインの有無を確認するだけで、「自分のデータだけ」を保証するのはアプリの責任です。

### FileBucket — 添付ファイルを扱う

`FileBucket` はファイルストレージの Block です。ローカルでは `.bb-data` に保存され、AWS では S3 になります。

ファイルアップロードは、**ブラウザが直接アップロードできる署名付き URL（presigned URL）** を発行する方式が基本です。バックエンドは URL を返すだけで、ファイル本体はバックエンドを経由しません。

```typescript
import { FileBucket } from '@aws-blocks/blocks';

const attachments = new FileBucket(scope, 'attachments');
```

```typescript
  // ApiNamespace のメソッドとして追加
  async getAttachmentUploadUrl(filename: string) {
    const user = await auth.requireAuth(context);
    // キーにユーザーごとのプレフィックスを付け、特殊文字は encode する
    const key = `${user.userSub}/${Date.now()}-${encodeURIComponent(filename)}`;
    const url = await attachments.putUrl(key, { expiresIn: 600 }); // 10 分有効
    return { key, url };
  },
```

返ってきた `key` を、チケットの `attachment_key` カラムに保存しておけば、後でダウンロード URL を発行できます。

```typescript
  async getAttachmentDownloadUrl(key: string) {
    await auth.requireAuth(context);
    return { url: await attachments.getUrl(key, { expiresIn: 600 }) };
  },
```

:::message
`attachments.get(path)` でファイル本体を読むこともできますが、**存在しないときは例外ではなく `null` が返る**点に注意してください。キーに `/` 以外の特殊文字（メールアドレスなど）が入るときは `encodeURIComponent` で包むのが安全です。
:::

#### 責務の境界（Auth / FileBucket / アプリ）

ここは AWS Blocks を理解するうえで大事なポイントです。`FileBucket` は **presigned URL を発行する Block** ですが、「このユーザーがこのファイルを取得してよいか」という**所有者判定は自動では行いません**。`auth.requireAuth(context)` も、あくまで**ログイン済みであることを確認するだけ**です。チケットや添付ファイルの所有者チェックは**アプリ側の責任**です。

役割を整理すると次のとおりです。

- **`AuthCognito`** … 認証（ログインしているか、誰か）を担当する
- **`FileBucket`** … ファイル保存と presigned URL の発行を担当する
- **アプリ（API handler）** … 「この人がこのリソースを触ってよいか」の認可判定を担当する

そのため、ダウンロード URL を発行するときは、**クライアントから渡された `key` をそのまま信用しない**のが安全です。チケット ID から DB を引き、`owner_sub = user.userSub` を確認したうえで、保存済みの `attachment_key` から URL を発行する、という流れにすると責務の分け方が分かりやすくなります。

```typescript
  // 責務境界を意識した安全な発行例（key を直接信用しない）
  async getTicketAttachmentUrl(ticketId: string) {
    const user = await auth.requireAuth(context);
    const ticket = await db.queryOne<Ticket>(
      sql`SELECT * FROM tickets WHERE id = ${ticketId} AND owner_sub = ${user.userSub}`
    );
    if (!ticket?.attachment_key) return { url: null };
    return { url: await attachments.getUrl(ticket.attachment_key, { expiresIn: 600 }) };
  },
```

:::message
「ログイン済み = 全ファイルにアクセス可」ではありません。`requireAuth` の後ろに、必ず DB の `owner_sub` チェックを置くのがポイントです。
:::

### Logger / Metrics / Dashboard — 観測する

最後に観測まわりです。3 つとも `(scope, id, options)` の形で作れます。

```typescript
import { Logger, Metrics, Dashboard } from '@aws-blocks/blocks';

const log = new Logger(scope, 'log');
const metrics = new Metrics(scope, 'metrics');
```

`Logger` は構造化ログ（JSON）を、`Metrics` はカスタムメトリクスを出します。**どちらも同期 API（`await` 不要）** です。ローカルでは標準出力に、AWS では CloudWatch（Metrics は EMF 経由）に流れます。

チケット作成時にメトリクスとログを仕込んでみましょう。

```typescript
  async createTicket(title: string, body: string, priority: TicketPriority = 'normal') {
    const user = await auth.requireAuth(context);
    const id = newId();
    await db.execute(sql`
      INSERT INTO tickets (id, owner_sub, title, body, priority)
      VALUES (${id}, ${user.userSub}, ${title}, ${body}, ${priority})
    `);
    metrics.emit('RequestCreated', 1, { unit: 'Count' }); // カスタムメトリクス
    log.info('ticket created', { id, priority });          // 構造化ログ
    return await db.queryOne<Ticket>(sql`SELECT * FROM tickets WHERE id = ${id}`);
  },
```

`Dashboard` は CloudWatch ダッシュボードを自動生成する Block です。`Logger` / `Metrics` のインスタンスを渡すと、そこから名前空間やロググループを読み取って配線してくれます。

```typescript
const dashboard = new Dashboard(scope, 'dashboard', {
  logger: log,
  metrics,
  metricConfigs: [{ name: 'RequestCreated', stat: 'Sum' }],
});
```

:::message
`Dashboard` は **CloudWatch 専用**です。ローカルモックでは実体を持たず、`url` は `null`、ダッシュボードへのリダイレクトは「先にデプロイしてね」という旨を返します。実際の見た目を確認するのは sandbox / 本番です。`Logger` と `Metrics` の出力自体はローカルのターミナルで確認できます。
:::

## 動作確認

`npm run dev` を起動し、ブラウザで `http://localhost:3000` を開きます。

1. テンプレートのサインアップフォームからアカウントを作成します（メールアドレス + 8 文字以上のパスワード）。
2. 確認コードを求められたら、**`npm run dev` を実行しているターミナル**を見てください。`[auth] xxx@example.com の確認コード: 123456` と出力されています。それを入力します。
3. サインインすると、チケットを作成・一覧できます。
4. チケットを作成すると、ターミナルに `RequestCreated` メトリクスのログと `ticket created` のログが出ます。

フロントエンドのコード（フォームや一覧 UI）はサンプルリポジトリ `api-namespace/` の `src/` を参照してコピーしてください。バックエンドの呼び出しは、すべて `import { api, authApi } from 'aws-blocks'` 経由です。

未ログインで `createTicket` を呼ぶと、次のように `401` が返ることも確認できます。

```json
{"error":{"code":401,"message":"Authentication required","data":{"name":"NotAuthenticatedException"}}}
```

## この章でできたこと

- `AuthCognito` でサインアップ〜サインインのフローを実装し、API をメソッド単位で保護した
- `Database` をマイグレーション付きで定義し、チケットの CRUD を `sql` テンプレートで書いた
- `FileBucket` で署名付き URL を使った添付ファイルの仕組みを作った
- `Logger` / `Metrics` / `Dashboard` で観測を組み込んだ（Dashboard はクラウド専用）

## 完了条件

- サインアップ、確認コード入力、サインインができる
- ログイン後にチケットを作成できる
- 自分のチケット一覧を取得できる
- `Logger` / `Metrics` の出力をローカルターミナルで確認できる
- 添付ファイルの upload URL / download URL を発行できる
- `Dashboard` はローカルでは実体を持たず、実 AWS で確認するものだと理解できる

## 次の章でやること

次の章では、リクエスト・レスポンスの外で動く処理を足します。`AsyncJob` でチケット作成通知を非同期化し、`CronJob` で open チケット数を定期集計し、`EmailClient` でメール通知を送ります。
