---
title: "[おまけ] Ollama でローカル LLM を使う"
free: true
---

:::message
この章はおまけです。AWS アカウントや Bedrock を使わず、canned よりも「LLM らしい」応答をローカルで確認したい場合に試してください。マシンスペックや Ollama の状態によって動作に差が出ます。**公開時点では Ollama 実行は未検証**です（手順紹介）。Ollama が起動していない場合、`Agent` は canned にフォールバックします。
:::

## この章でやること

第7章では、ローカルの `Agent` は canned provider（擬似応答）でした。ここでは `Agent` を **Ollama**（ローカルで動く LLM ランタイム）に向けて、実際の生成を手元で試します。AWS は不要です。

## 前提

- 第7章までの実装
- ある程度のメモリ／VRAM（モデルサイズによる）

## 実装

### Ollama のインストールと起動

[Ollama](https://ollama.com) をインストールし、サーバーを起動します。

```bash
# macOS なら Homebrew でも入ります
brew install ollama

# サーバーを起動（OpenAI 互換エンドポイントが http://localhost:11434 で待ち受ける）
ollama serve
```

別のターミナルでモデルを pull します。ここでは軽量な `llama3.1:8b` を使います。

```bash
ollama pull llama3.1:8b
```

Ollama は **OpenAI 互換エンドポイント** を `http://localhost:11434/v1` で提供します。AWS Blocks の `Agent` はこれを利用します。

### Agent の local provider を設定する

`Agent` の `model` は `{ deployed, local? }` という形です。`deployed` はデプロイ時（sandbox / 本番）に使うモデル、`local` はローカル開発で使うモデルです。`local` を指定すると、ローカルで canned の代わりにそのモデルが使われます。

Ollama は `provider: 'openai-api'` として、エンドポイントを Ollama に向けます。プリセット `OllamaModels` を使うのが簡単です。

```typescript
import { Agent, BedrockModels, OllamaModels } from '@aws-blocks/blocks';

const agent = new Agent(scope, 'support', {
  model: {
    deployed: BedrockModels.DEFAULT,  // sandbox / 本番では Bedrock
    local: OllamaModels.SMALL,        // ローカルでは Ollama の llama3.1:8b
  },
  inferenceOnly: true,
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

`OllamaModels.SMALL` は次の設定と同じです。自分で書く場合はこう指定します。

```typescript
local: {
  provider: 'openai-api',
  modelId: 'llama3.1:8b',
  endpoint: 'http://localhost:11434/v1',
  apiKey: 'ollama',
},
```

:::message
provider の値は `'bedrock' | 'openai-api' | 'canned'` の 3 つです。**`'ollama'` という provider 値はありません**。Ollama は OpenAI 互換なので `'openai-api'` を使い、エンドポイントを Ollama に向けます。vLLM など他の OpenAI 互換サーバーも同じ方法で使えます。
:::

`OllamaModels` には用途に応じたプリセットがあります。

| プリセット | モデル | 目安 |
|---|---|---|
| `XSMALL` | llama3.2:3b | 軽量・高速 |
| `SMALL` | llama3.1:8b | バランス型 |
| `MEDIUM` | deepseek-r1:14b | 推論強め |

## 動作確認

Ollama サーバー（`ollama serve`）を起動した状態で `npm run dev` を起動し、第7章と同じ `draftAnswer` を呼びます。canned の固定文ではなく、Ollama のモデルが生成した回答案が返れば成功です。

`Agent` はモデルの利用可否をヘルスチェックします。Ollama が起動していない／モデルが pull されていない場合は利用できないと判断され、最終フォールバックとして canned に戻ります。「canned の応答が返ってくる」ときは、まず `ollama serve` とモデルの pull を確認してください。

## この章でできたこと

- Ollama をローカルで起動し、モデルを pull した
- `Agent` の `model.local` を Ollama（`openai-api` 互換）に向けた
- canned ではなく実モデルの生成をローカルで確認する手順を理解した（**公開時点では未検証**）

## 完了条件

- Ollama を起動できる
- 対象モデルを pull できる
- `Agent` の `model.local` を `openai-api` provider として Ollama に向けられる
- Ollama が使えない場合は canned にフォールバックすることを理解できる

> 「canned ではなく Ollama モデルによる応答が返る」ことの確認は公開時点で未検証のため、完了条件には含めていません。

## 次の章でやること

次が最終章です。ここまで作ってきたものを振り返り、AWS Blocks と CDK layer の使いどころ、local / sandbox / production の違いを整理し、発展課題を紹介します。
