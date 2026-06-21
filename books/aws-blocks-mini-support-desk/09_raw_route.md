---
title: "[おまけ] JSON-RPC ではなく RawRoute で実装する"
free: true
---

## この章でやること

ここまで、API はすべて `ApiNamespace` で書いてきました。`ApiNamespace` は **JSON-RPC**（共有エンドポイント `POST /aws-blocks/api` にメソッド名と引数を JSON で送る方式）で、フロントからは `import { api } from 'aws-blocks'` で**型安全に**呼べました。

AWS Blocks には、もう 1 つ別の API の公開方法があります。**`RawRoute`（生 HTTP ルート）** です。任意の HTTP メソッド・パス・ステータス・ヘッダ・ボディを自分で扱える、素朴な HTTP エンドポイントです。

この章では、**「Mini Support Desk の API を丸ごと `RawRoute` で実装し直すとどうなるか」** を、`ApiNamespace` との対比で見ていきます。両者の違い・置き換え方・使い分けを整理するのが目的です。

:::message
この章はおまけです。ローカルモックだけで完結します（AWS 不要）。
**フル実装はサンプルリポジトリの [`raw-route/`](https://github.com/tkhashi/aws-blocks-mini-support-desk/tree/main/raw-route) ディレクトリにあります。** リポジトリは、これまで作ってきた `ApiNamespace` 版の [`api-namespace/`](https://github.com/tkhashi/aws-blocks-mini-support-desk/tree/main/api-namespace) と、この章の RawRoute 版 `raw-route/` の、**型情報もデータも共有しない 2 つの独立プロジェクト**に分かれています。
:::

## ApiNamespace と RawRoute の違い

まず全体像です。

| | `ApiNamespace`（JSON-RPC） | `RawRoute`（生 HTTP） |
|---|---|---|
| プロトコル | JSON-RPC 2.0 | 任意の HTTP |
| エンドポイント | 共有の `POST /aws-blocks/api` | 自分で決めたパス（例 `GET /tickets/{id}`） |
| HTTP メソッド | POST 固定 | `GET`/`POST`/`PUT`/`DELETE`/… 自由 |
| リクエスト | メソッド名＋引数（自動デシリアライズ） | `context.request` から自分で読む |
| レスポンス | 戻り値を自動でシリアライズ | `context.response.send()` で自分で返す |
| クライアント呼び出し | `import { api } from 'aws-blocks'` | 素の `fetch(path)` |
| 型安全 | フロント↔バックで**型が繋がる** | 繋がらない（自分で型を合わせる） |
| 認証 | `auth.requireAuth(context)`（opt-in） | `auth.requireAuth(context)`（opt-in・**同じ**） |
| ルーティング単位 | メソッド | メソッド＋パス |
| 向く用途 | アプリ内のフロント↔バック API | webhook・ファイルDL・リダイレクト・外部公開 URL |

ポイントは、**`RawRoute` は HTTP の生の表現を完全に手に握る**ことです。その代わり、`ApiNamespace` が自動でやってくれていた「シリアライズ」「型の接続」「配線」を自分で書く必要があります。

## RawRoute の基本形

`RawRoute` は `new RawRoute(scope, id, { method, path, handler })` で作ります。**構築するだけでルートが登録される**ので、`ApiNamespace` のように `export` する必要はありません（クライアントは `fetch` で叩くだけなので、エクスポートしても意味がありません）。

```typescript
import { RawRoute } from '@aws-blocks/blocks';

new RawRoute(scope, 'health', {
  method: 'GET',
  path: '/health',
  handler: async (context) => {
    context.response.status = 200;
    context.response.headers.set('Content-Type', 'text/plain');
    context.response.send('ok');
  },
});
```

`handler` が受け取る `context` は、`ApiNamespace` のメソッドが受け取るものと**同じ `BlocksContext`** です。

- `context.request` … `headers` / `json()` / `text()` / `body`（ストリーム）/ `url`（クエリ解析用の `URL`）/ `params`（`/tickets/{id}` の `{id}`）/ `signal`
- `context.response` … `status` / `headers` / `send(body)`（文字列でもオブジェクトでも可。オブジェクトは JSON 化される）

:::message
パスは `path` で明示するか、省略すると scope チェーンから導出されます。構造を変えると導出パスも変わるため、**安定させたいルートは明示**しましょう。また `/aws-blocks` 配下は `ApiNamespace` 用の予約パスなので使えません。
:::

## 認証を RawRoute に置き換える

これまでフロントは `<Authenticator>` という組み込み UI に `authApi`（= `auth.createApi()`）を渡すだけでした。RawRoute 版では、この組み込みをやめ、**`AuthCognito` のメソッドを RawRoute から直接呼びます**。

```typescript
// サインイン（成功すると AuthCognito がセッション Cookie を自動で発行する）
new RawRoute(scope, 'auth-sign-in', {
  method: 'POST',
  path: '/auth/sign-in',
  handler: async (context) => {
    const { username, password } = await context.request.json();
    const result = await auth.signIn(username, password, context);
    context.response.send(result);
  },
});
```

`auth.signIn(username, password, context)` は、成功時に **HttpOnly のセッション Cookie** を `context` 経由でレスポンスにセットします。`signUp` / `confirmSignUp` / `signOut` / `getCurrentUser` も同様に `context` を取ります。つまり**トークンを手で管理する必要はありません**。クライアントは `fetch` に `credentials: 'include'` を付けるだけで、Cookie が自動的に往復します。

```typescript
// フロント側（素の fetch）
await fetch('/auth/sign-in', {
  method: 'POST',
  credentials: 'include',                 // ← Cookie を送受信する
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ username, password }),
});
```

保護したいエンドポイントでは、`ApiNamespace` のときとまったく同じように `auth.requireAuth(context)` を呼びます。未ログインなら例外が投げられ、**RawRoute のディスパッチャがそれを自動で HTTP 401 に変換**します。

```typescript
new RawRoute(scope, 'tickets-create', {
  method: 'POST',
  path: '/tickets',
  handler: async (context) => {
    const user = await auth.requireAuth(context);  // 未ログインなら 401
    const { title, body, priority = 'normal' } = await context.request.json();
    const id = newId();
    await db.execute(sql`
      INSERT INTO tickets (id, owner_sub, title, body, priority)
      VALUES (${id}, ${user.userSub}, ${title}, ${body}, ${priority})
    `);
    context.response.status = 201;
    context.response.send(await db.queryOne<Ticket>(sql`SELECT * FROM tickets WHERE id = ${id}`));
  },
});
```

`owner_sub` でアクセス制御するのも `ApiNamespace` 版と同じです。**認証まわりの考え方は変わらず、API の公開方法だけが変わる**わけです。

## API 全体の対応関係

`ApiNamespace` のメソッドは、そのまま「メソッド＋パス」の RawRoute に 1 対 1 で置き換えられます。

| `ApiNamespace`（`api-namespace/`） | RawRoute（`raw-route/`） |
|---|---|
| `api.createTicket(...)` | `POST /tickets` |
| `api.listTickets()` | `GET /tickets` |
| `api.getTicket(id)` | `GET /tickets/{id}` |
| `api.closeTicket(id)` | `POST /tickets/{id}/close` |
| `api.draftAnswer(id)` | `POST /tickets/{id}/draft-answer` |
| `api.getAttachmentUploadUrl(name)` | `POST /attachments/upload-url` |
| `api.searchDocs(q)` | `GET /search?q=...` |
| `authApi`（`<Authenticator>`） | `POST /auth/sign-up` `/auth/confirm` `/auth/sign-in` `/auth/sign-out`・`GET /auth/me` |

RPC では「メソッド名」で呼び分けていたものが、HTTP では「メソッド＋パス」になります。一覧取得が `GET`、作成が `POST`、URL のパスでリソースを表す——という **REST 的な設計**が自然に書けるのが RawRoute の特徴です。

## RawRoute ならではの例：CSV エクスポート

RawRoute の価値が一番分かりやすいのは、**JSON 以外を返したいとき**です。たとえばチケット一覧の CSV ダウンロード。JSON-RPC では「ファイルとしてダウンロードさせる」のが不自然ですが、RawRoute なら `Content-Type` と `Content-Disposition` を自分で設定するだけです。

```typescript
new RawRoute(scope, 'export-tickets-csv', {
  method: 'GET',
  path: '/export/tickets.csv',
  handler: async (context) => {
    const user = await auth.requireAuth(context);
    const tickets = await db.query<Ticket>(
      sql`SELECT * FROM tickets WHERE owner_sub = ${user.userSub} ORDER BY created_at DESC`
    );
    const csv = toCsv(tickets); // ヘッダ行＋各行を組み立てる

    context.response.headers.set('Content-Type', 'text/csv; charset=utf-8');
    context.response.headers.set('Content-Disposition', 'attachment; filename="tickets.csv"');
    context.response.send(csv);  // 文字列はそのままボディになる
  },
});
```

ほかにも、外部サービスからの **webhook 受信**（固定 URL に POST される）、OAuth の**コールバック**、**リダイレクト**（`302` + `Location`）、ヘルスチェックなど、「HTTP の契約が決まっている入り口」は RawRoute の出番です。

## 動作確認（ローカル）

`raw-route/` ディレクトリで `npm install && npm run dev` を起動すれば、すべてローカルモックで確認できます。`curl` でセッション Cookie を保存しながら一連の流れを叩けます。

```bash
# サインアップ → 確認コード（ターミナルに出力）→ サインイン（Cookie をjarに保存）
curl -c jar -X POST localhost:3001/auth/sign-up  -H 'Content-Type: application/json' -d '{"username":"me@example.com","password":"Passw0rd!23"}'
curl -c jar -X POST localhost:3001/auth/confirm  -H 'Content-Type: application/json' -d '{"username":"me@example.com","code":"<ターミナルのコード>"}'
curl -c jar -X POST localhost:3001/auth/sign-in  -H 'Content-Type: application/json' -d '{"username":"me@example.com","password":"Passw0rd!23"}'

# 以降は Cookie を送る（-b jar）
curl -b jar -X POST localhost:3001/tickets -H 'Content-Type: application/json' -d '{"title":"test","body":"hello","priority":"normal"}'
curl -b jar localhost:3001/tickets
curl -b jar localhost:3001/export/tickets.csv      # text/csv が返る
curl       localhost:3001/tickets                  # Cookie なし → 401
```

フロントエンドは vite の proxy で `:3001` の API に転送しているため、ブラウザからは同一オリジンとして Cookie が機能します。画面の使い勝手は `main` とほぼ同じで、裏側の呼び出しだけが `fetch` に変わっています。

## RawRoute で失うもの・得るもの

最後に、置き換えで何が変わるかを整理します。

**失うもの（= `ApiNamespace` の利点）**

- **end-to-end の型安全** … `api.createTicket(...)` の引数・戻り値の型がフロントまで繋がっていたが、`fetch` では自分で型を合わせる必要がある。
- **自動の配線・ボイラープレート削減** … シリアライズ／デシリアライズ、エンドポイント解決を自分で書くことになる。

**得るもの（= `RawRoute` の利点）**

- **HTTP の完全な制御** … メソッド・パス・ステータス・ヘッダ・非 JSON ボディを自由に扱える。
- **外部との契約に強い** … webhook・ファイルダウンロード・リダイレクト・第三者が叩く固定 URL を素直に表現できる。

結論はシンプルです。**アプリ内のフロント↔バック API は `ApiNamespace`（型安全で楽）、外部に向けた HTTP の入り口や非 JSON が必要な所は `RawRoute`**。両者は排他ではなく、1 つのバックエンドに混在させられます（この章は「全部 RawRoute にするとどうなるか」をあえて試した、という位置づけです）。

## この章でできたこと

- `ApiNamespace`（JSON-RPC）と `RawRoute`（生 HTTP）の違いを整理した
- 認証を含む全 API を `RawRoute` で実装し直すと何がどう変わるかを理解した（フル実装はサンプルの `raw-route/` ディレクトリ）
- 認証は `AuthCognito` のメソッド＋セッション Cookie で再現でき、`requireAuth` の使い方は `ApiNamespace` と同じだと理解した
- CSV エクスポートのような非 JSON レスポンスが `RawRoute` で素直に書けることを確認した

## 完了条件

- `RawRoute` が「メソッド＋パス＋handler」で HTTP エンドポイントを作る Block だと説明できる
- `context.request` / `context.response` で生のリクエスト・レスポンスを扱えると理解できる
- 認証が Cookie ＋ `requireAuth` で `ApiNamespace` 版と同じ考え方で書けると理解できる
- `ApiNamespace` と `RawRoute` の使い分け（型安全 vs HTTP 制御）を説明できる

## 次の章でやること

次が最終章です。ここまで作ってきたものを振り返り、AWS Blocks と CDK layer の使いどころ、local / sandbox / production の違いを整理し、発展課題を紹介します。
