---
title: "[任意・課金あり] Bedrock RAG — ドキュメント検索と回答案生成"
free: true
---

## この章でやること

問い合わせ内容に対して、参照ドキュメントを基に AI が回答案を作る **RAG**（Retrieval-Augmented Generation）を作ります。

1. 参照ドキュメントを用意する
2. **`KnowledgeBase`** でドキュメントを検索する（ローカルは簡易検索）
3. **`Agent`** で回答案を生成する（ローカルは canned 擬似応答）
4. **sandbox** で実 Bedrock を使って動かす

全体像はこうです。

```text
ticket.body
  ↓
KnowledgeBase で関連ドキュメント検索
  ↓
Agent が回答案を生成
  ↓
画面に表示
```

:::message alert
この章の **sandbox / Bedrock 実行**は AWS アカウント・Bedrock のモデルアクセス・課金が必要です。**公開時点では未検証**（手順紹介）です。ローカル部分（`KnowledgeBase` の TF-IDF 検索・`Agent` の canned 応答）は AWS なしで確認できます。
:::

第3〜4章まではローカルモックだけで動きました。RAG も**ローカルで開発・確認**できます。実 Bedrock での確認だけ sandbox を使います。

## 前提

- 第4章までの実装（ローカル部分）
- 実 Bedrock を使う節のみ AWS アカウントと Bedrock のモデルアクセス

## ローカルと sandbox の違い

最初に、この章で一番大事な「ローカルと sandbox の差」を押さえます。

```text
ローカル（KnowledgeBase / Agent）:
  KnowledgeBase … TF-IDF による簡易キーワード検索
  Agent        … canned provider（キーワードベースの擬似応答）
  AWS 不要・課金なし

sandbox:
  KnowledgeBase … 実 Bedrock KnowledgeBase（Titan 埋め込み + ベクトル検索）
  Agent        … 実 Bedrock のモデル
```

**ローカルの KnowledgeBase は埋め込み検索ではなく TF-IDF** です。つまり「意味が近い」ではなく「単語が一致する」検索です。動作確認には十分ですが、本物の Bedrock の検索結果とスコアは一致しません（後述の注意点も参照）。

## 実装

### 参照ドキュメントを用意する

`KnowledgeBase` の `source` に指定したフォルダのドキュメントが検索対象になります。`docs/knowledge-base/` を作り、Markdown を置きます。

```markdown:docs/knowledge-base/faq.md
# よくある質問 (FAQ)

## パスワードをリセットするには
ログイン画面の「パスワードをお忘れですか」から、登録メールアドレスを入力してください。
確認コードが届いたら、新しいパスワードを設定できます。

## VPN に接続できない
社外から VPN に接続できない場合は、まずネットワーク接続を確認し、
VPN クライアントを最新版に更新してください。
```

同様に `support-policy.md`（サポート対応ポリシー）など、必要なドキュメントを追加します。サブフォルダ名は `metadata.folder` として自動的に付与され、検索時のフィルタに使えます。

### KnowledgeBase で検索する

```typescript
import { KnowledgeBase } from '@aws-blocks/blocks';

const kb = new KnowledgeBase(scope, 'docs', {
  source: './docs/knowledge-base',
});
```

検索は `retrieve()` の 1 メソッドです。

```typescript
  async searchDocs(query: string) {
    await auth.requireAuth(context);
    return await kb.retrieve(query, { maxResults: 3 });
  },
```

`retrieve` は `{ text, score, source, metadata }` の配列を返します。ローカルでは初回検索時にドキュメントを段落単位でチャンク化し、`.bb-data` にキャッシュします。

:::message alert
**日本語とローカル TF-IDF の相性に注意。** ローカルの簡易検索は空白区切りの単語一致が基本です。日本語は単語が空白で区切られないため、たとえばドキュメントの「パスワードリセット」（空白なし）はクエリ「リセット 方法」とマッチしないことがあります。`VPN` のような英単語はマッチします。
これは**ローカルモックの制約**で、sandbox の実 Bedrock は Titan 埋め込みにより日本語でも意味ベースで検索できます。ローカルでは「英単語やドキュメント中の語句をそのまま含むクエリ」で動作確認するとよいでしょう。
:::

### Agent で回答案を生成する

`Agent` は AI エージェントの Block です。ツール呼び出し・会話・ストリーミングに対応します。ここでは `KnowledgeBase` を検索する `searchDocs` ツールを持たせ、回答案を作らせます。

```typescript
import { Agent, BedrockModels } from '@aws-blocks/blocks';
import { z } from 'zod';

const agent = new Agent(scope, 'support', {
  model: { deployed: BedrockModels.DEFAULT },
  inferenceOnly: true, // 会話履歴の永続化インフラを省略（単発の回答案生成）
  systemPrompt: 'あなたは丁寧なサポート担当です。参考ドキュメントを基に回答案を作成してください。',
  tools: (tool) => ({
    searchDocs: tool({
      description: 'サポートドキュメントを検索する',
      parameters: z.object({ query: z.string() }),
      handler: async ({ input }) => kb.retrieve(input.query, { maxResults: 3 }),
    }),
  }),
});
```

ポイント:

- **`model.deployed`** に Bedrock のモデルを指定します（`BedrockModels.DEFAULT` など）。
- **ローカルでは canned provider が自動的に使われます**。ローカルにモデルを指定しなければ、AWS を呼ばずにキーワードベースの擬似応答が返ります。だから AWS なしで開発できます。
- **`KnowledgeBase` 連携**は、`Agent` の設定項目ではなく**ツールの中で `kb.retrieve` を呼ぶ**形で行います。

回答案を生成する API を追加します。`Agent.stream()` を呼び、サーバー側では `complete()` で完了を待って全文を取り出します。

```typescript
  async draftAnswer(ticketId: string) {
    const user = await auth.requireAuth(context);
    const ticket = await db.queryOne<Ticket>(
      sql`SELECT * FROM tickets WHERE id = ${ticketId} AND owner_sub = ${user.userSub}`
    );
    if (!ticket) throw new Error('ticket not found');

    const result = await agent.stream(
      `件名: ${ticket.title}\n本文: ${ticket.body}\nこの問い合わせへの回答案を作成してください。`
    );
    const done = await result.complete();
    return { answer: done.text ?? '' };
  },
```

## 動作確認（ローカル）

`npm run dev` を起動し、ログインしてチケットを作成してから `draftAnswer` を呼びます。ローカルでは canned provider のため、次のような擬似応答が返ります。

```json
{ "answer": "This is a canned mock response. No real model was called. [canned] " }
```

これは「配線が正しく、Agent が呼べている」ことの確認です。`searchDocs` を英単語（例: `VPN`）で叩くと、該当する FAQ のチャンクが返ることも確認できます。**本物らしい応答**を見たい場合は、次章の Ollama か、この後の sandbox を使います。

## sandbox で Bedrock を実行する

:::message alert
ここからは実 Bedrock を使います。AWS アカウントが必要で、**モデル利用・埋め込み・ベクトル検索に費用が発生**します。章末の費用・削除を確認してください。
:::

### Bedrock のモデルアクセスを有効化する

Amazon Bedrock は、使うモデルを**リージョンごとに事前有効化**する必要があります。

1. AWS コンソールで Bedrock を開き、リージョン（例: `us-east-1`）を確認する
2. **Model access** から、使うモデル（埋め込み用 Titan、生成用の Claude など）へのアクセスをリクエスト・有効化する

### デプロイして確認する

```bash
npm run sandbox
```

デプロイ時に `KnowledgeBase` は `source` フォルダのドキュメントを S3 に同期し、取り込み（ingestion）を開始します。取り込みは非同期なので、完了まで少し待ちます。

確認観点:

1. `searchDocs` を**日本語の自然な問い合わせ**で叩くと、ローカル（TF-IDF）と違って意味ベースで関連チャンクが返る
2. `draftAnswer` が canned ではなく、実モデルによる回答案を返す
3. ローカルモックとの差（スコアの違い・日本語検索の精度）を体感する

## 費用の目安

- **Bedrock のモデル呼び出し** … 入力・出力トークン課金。回答生成のたびに少額が発生します。
- **KnowledgeBase（埋め込み + ベクトルストア）** … ドキュメント取り込み時の埋め込み生成と、検索時の課金があります。
- **Aurora（Database Block）** … 前章同様、起動中は時間課金。

検証用途なら少額ですが、**使い終わったら必ず削除**してください。正確な料金は[料金ページ](https://aws.amazon.com/jp/pricing/)で確認します。

## リソースの削除

```bash
npm run sandbox:destroy
```

KnowledgeBase・ベクトルストア・Aurora などがまとめて削除されます。AWS コンソールでスタックが消えていることを確認しましょう。

## この章でできたこと

- 参照ドキュメントを用意し、`KnowledgeBase` で検索した（ローカルは TF-IDF・日本語の制約を理解）
- `Agent` に検索ツールを持たせ、回答案を生成した（ローカルは canned）
- sandbox で実 Bedrock を有効化し、埋め込み検索と生成を確認する流れを理解した（**公開時点では未検証**）
- ローカルモックと実 Bedrock の差を理解した

## 完了条件

ローカル（AWS 不要）で次が満たせていれば、この章の本編部分は完了です。

- ローカルで `KnowledgeBase.retrieve()` を呼び出せる
- ローカルの `Agent` が canned 応答を返すことを確認できる
- ローカル KnowledgeBase は TF-IDF ベースの簡易検索であり、実 Bedrock の意味検索とは違うと説明できる
- sandbox 実行は任意・課金ありであることを理解できる

> sandbox / Bedrock の実行（取り込み完了・canned ではない回答案）は公開時点で未検証のため、完了条件には含めていません。実行する場合の確認観点は本文「sandbox で Bedrock を実行する」を参照してください。

## 次の章でやること

次の章は「おまけ」です。AWS や Bedrock を使わずに、より LLM らしい応答をローカルで確認するために、`Agent` を **Ollama** に向ける方法を紹介します。
