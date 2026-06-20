---
title: "[本編] プロジェクトを作成してローカル起動する"
free: true
---

## この章でやること

AWS Blocks プロジェクトを新規作成し、`npm run dev` でローカル起動します。生成されるディレクトリ構成を読み解き、AWS Blocks の骨格である `Scope` と `ApiNamespace` を理解します。この章はすべてローカルモックで完結し、**AWS アカウントは不要**です。

## 前提

- Node.js 22 以上
- npm 10 以上

AWS Blocks 公式リポジトリおよび Developer Guide の前提に合わせています。先にバージョンを確認しておきましょう。

```bash
node --version
npm --version
```

## 実装

### プロジェクト作成

`create-blocks-app` でプロジェクトを生成します。テンプレートはいくつかありますが、ここでは React + Vite の `react` テンプレートを使います。

```bash
npx @aws-blocks/create-blocks-app mini-support-desk --template react
cd mini-support-desk
npm install
```

:::message
テンプレートには `default`（リアルタイム ToDo アプリ）, `bare`, `react`, `backend`（フロントなし）, `nextjs`, `auth-cognito`, `amplify`, `demo` があります。この Book は `react` テンプレートを起点に進めます。
:::

:::message
公式 Getting Started では `npm create @aws-blocks/blocks-app@latest` が使われています。この Book は `create-blocks-app` README にある `npx @aws-blocks/create-blocks-app` 形式で進めます。どちらも AWS Blocks プロジェクトを作成するための入口です。
:::

### npm install

依存関係を確実に揃えるため、この Book ではプロジェクト作成後に**必ず `npm install` を実行**します（上のコマンドにも含めています）。手動でクローンした場合も同様です。

```bash
npm install
```

バックエンドの Block は **`@aws-blocks/blocks`** という 1 つのパッケージに含まれています。`AuthCognito` も `Database` も、すべてここからインポートします。

:::message
AWS Blocks は Preview のため、バージョンによって API や生成テンプレートが変わる可能性があります。サンプルリポジトリでは `@aws-blocks/blocks` を `package-lock.json` で固定しています（第1章「検証環境」参照。検証時点で `@aws-blocks/blocks@0.1.2`）。手元のバージョンと挙動が違う場合は、固定バージョンを基準に確認してください。
:::

### ディレクトリ構成を理解する

生成された主なファイルはこうなっています。

```text
mini-support-desk/
├── aws-blocks/
│   ├── index.ts          # バックエンド本体（API・Block 定義）
│   ├── index.cdk.ts      # CDK スタック定義（CDK layer はここに足す）
│   ├── index.handler.ts  # Lambda ハンドラ（自動生成・触らない）
│   ├── client.js         # フロント用クライアント（自動生成・触らない）
│   └── scripts/          # dev / sandbox / deploy などのスクリプト
├── src/                  # フロントエンド（React + Vite）
│   ├── main.tsx
│   └── App.tsx
├── cdk.json
├── package.json
└── vite.config.ts
```

#### `aws-blocks/index.ts`

バックエンドの中心です。ここに Block を定義し、API を `export` します。初期状態はこんな雰囲気です（テンプレートの内容を Mini Support Desk 用に置き換えていきます）。

```typescript
import { ApiNamespace, Scope } from '@aws-blocks/blocks';

// Scope はバックエンドのリソース境界。アプリ名を付ける。
const scope = new Scope('mini-support-desk');

// ApiNamespace は型安全な API。第3引数のファクトリが返す
// 各 async メソッドが、そのまま 1 つの API エンドポイントになる。
export const api = new ApiNamespace(scope, 'api', (context) => ({
  async greet(name: string) {
    return { message: `こんにちは、${name} さん` };
  },
}));
```

ポイントは 2 つです。

- **`Scope`** … バックエンド全体の入れ物です。`new Scope('mini-support-desk')` のようにアプリ名を渡します。以降つくる Block はすべてこの `scope` にぶら下げます。
- **`ApiNamespace`** … `(context) => ({ ...メソッド })` というファクトリを受け取ります。返したオブジェクトの各メソッドが API になります。`context` は認証などに使うリクエスト情報です（第3章で使います）。

#### `aws-blocks/index.cdk.ts`

AWS にデプロイするときの CDK スタック定義です。Block だけで足りない AWS リソース（SQS など）を足したくなったら、このファイルに書きます（第5章）。普段は触りません。

#### `.bb-data`

`npm run dev` で動かすと、ローカルモックのデータが `.bb-data/` ディレクトリに保存されます。Database や FileBucket の中身はここに永続化されます。**状態をリセットしたいときはこのディレクトリを削除**すればまっさらになります。

```bash
rm -rf .bb-data   # ローカルの状態を全消去
```

`.bb-data` は `.gitignore` に入っているので、コミットされません。

### ローカル起動

```bash
npm run dev
```

`npm run dev` は次の 2 つを同時に立ち上げます。

- **フロントエンド**（Vite）: `http://localhost:3000`
- **バックエンドのローカル API サーバー**: `http://localhost:3001`

ブラウザで `http://localhost:3000` を開くと、テンプレートのアプリが表示されます。AWS アカウントなしで、すべてのモックがその場で動いています。

## 動作確認

フロントエンドからは、バックエンドの API を **`aws-blocks` という別名（エイリアス）** でインポートして、ふつうの関数のように呼べます。型もそのまま流れてきます。

```typescript
// src/ のどこか
import { api } from 'aws-blocks';

const res = await api.greet('世界');
// res.message は string 型として推論される
```

:::message
`import { api } from 'aws-blocks'`（バレ指定子）と、バックエンドの `import ... from '@aws-blocks/blocks'`（スコープ付き）は**別物**です。前者はフロントが API を呼ぶための自動生成クライアント、後者は Block 本体です。混同しないようにしましょう。
:::

裏側は **JSON-RPC 2.0** で通信していますが、これは AWS Blocks が隠してくれるので、私たちは型付きの関数呼び出しだけを意識すれば OK です。動作確認だけ手動でやりたいときは、次のように `curl` でも叩けます（普段は不要）。

```bash
curl -X POST http://localhost:3001/aws-blocks/api \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"api.greet","params":["世界"],"id":1}'
# → {"jsonrpc":"2.0","result":{"message":"こんにちは、世界 さん"},"id":1}
```

`method` は `<export 名>.<メソッド名>`（ここでは `api.greet`）です。

## この章でできたこと

- `create-blocks-app` でプロジェクトを作り、`npm run dev` で起動した
- `Scope` と `ApiNamespace` の役割を理解した
- フロントは `aws-blocks` エイリアスから型安全に API を呼べることを確認した
- ローカルの状態は `.bb-data` に保存され、削除でリセットできることを理解した

## 完了条件

次がすべて満たせていれば、この章は完了です。

- `node --version` が 22 以上である
- `npm --version` が 10 以上である
- `npm install` が成功する
- `npm run dev` が起動する
- ブラウザで `http://localhost:3000` を開ける
- `api.greet` をフロントまたは `curl` から呼び出せる

## 次の章でやること

次の章では、Mini Support Desk の土台を一気に作ります。`AuthCognito` でログインを付け、`Database` で問い合わせチケットを保存し、`FileBucket` で添付ファイルを扱い、`Logger` / `Metrics` / `Dashboard` で観測できるようにします。
