---
title: "Mastra + Next.js でローカルLLMチャットアプリを作る"
---

## この章でやること

- Next.js + Mastra で**自作のチャットアプリ**を立ち上げる
- ローカルで動く Ollama モデルをエージェントの推論エンジンに使う
- AI SDK UI（`useChat` + AI Elements）でストリーミングチャット画面を作る

完成イメージ: ブラウザの `/chat` を開くと、メッセージ入力に対して**ローカル Ollama モデルがストリーミング応答**するチャット画面ができます。すべてローカルで完結し、外部 API キーは不要です。

---

:::message
**前提**
- Node.js 20 以降がインストールされていること（`node -v` で確認）
- Ch.02〜03 を終え、`ollama list` に `gemma4:e4b` などのモデルがあること
- Ollama が起動していること（`curl http://localhost:11434` で `Ollama is running`）

**コース別の使用モデル**
この章のサンプルは `gemma4:e4b` を使用します。他のコースの方は `.env` のモデル名を変更してください。

| コース | 使用モデル |
|---|---|
| Lite | `gemma4:e4b`（または `gemma4:e2b`） |
| Standard | `gemma4:e4b` または `qwen3.6:27b` |
| Pro | `qwen3.6:27b` または `gemma4:26b` |

`qwen3.6:27b` はツール呼び出しに対応しており、後半の発展課題で有利です。
:::

---

## Mastra とは

[Mastra](https://mastra.ai) は TypeScript 製の AI エージェントフレームワークです。
エージェント・ツール・メモリ・ワークフローを統一的な API で扱え、Vercel AI SDK と組み合わせてフロントエンドのストリーミング UI に直結できます。

本章は公式ガイド [Get started with Next.js](https://mastra.ai/guides/getting-started/next-js) をベースに、推論エンジンを **OpenAI ではなくローカル Ollama に差し替える** 構成で進めます。

完成版のコードは [tkhashi/mac-local-llm-mastra-sample](https://github.com/tkhashi/mac-local-llm-mastra-sample) にあります。詰まったときの参照や、`git clone` してそのまま動かすベースとして使えます。

---

## Step 1: Next.js プロジェクトを作成する

作業ディレクトリに移動して Next.js プロジェクトを作ります。

```bash
mkdir -p ~/local-llm-hands-on
cd ~/local-llm-hands-on

npx create-next-app@latest mastra-ollama-chat \
  --yes --ts --eslint --tailwind --src-dir --app --turbopack \
  --no-react-compiler --no-import-alias

cd mastra-ollama-chat
```

---

## Step 2: Mastra と AI SDK 関連パッケージをインストールする

```bash
npm install @mastra/core @mastra/ai-sdk
npm install ai@^5 @ai-sdk/react@^2 ollama-ai-provider-v2@^1.5.5 zod
```

| パッケージ | 役割 |
|---|---|
| `@mastra/core` | Mastra のエージェント・メモリ・ツール基盤 |
| `@mastra/ai-sdk` | Mastra と Vercel AI SDK の橋渡し |
| `@ai-sdk/react` | React 用フック（`useChat`） |
| `ai` | AI SDK 本体（ストリーミング・型） |
| `ollama-ai-provider-v2` | ローカル Ollama を AI SDK モデルとして使うためのプロバイダ |
| `zod` | エージェントの入出力スキーマ定義 |

:::message alert
**バージョン固定の理由**
`@mastra/ai-sdk` は AI SDK **v5** をベースにしています。`ai@latest` や `@ai-sdk/react@latest` をそのまま入れると v6 系・v3 系が入って整合性が崩れるため、上記のように `ai@^5` `@ai-sdk/react@^2` `ollama-ai-provider-v2@^1.5.5` を明示してください。
:::

続けて、AI SDK UI のコンポーネント（AI Elements）をプロジェクトに展開します。

```bash
npx ai-elements@latest add conversation message prompt-input
```

途中で `components.json` を作るか聞かれたら **Yes** で進めます。完了すると `src/components/ai-elements/` に `conversation.tsx` `message.tsx` `prompt-input.tsx` が、`src/components/ui/` に依存するコンポーネントが生成されます。

:::message alert
**`npx ai-elements@latest` を引数なしで実行しないでください。**
全コンポーネントが入り、未使用ファイルの型エラーで `npm run build` が失敗します。本章で使う 3 つだけを `add` で指定するのが正解です。
:::

---

## Step 3: 環境変数を設定する

プロジェクトルートに `.env` を作ります。

```bash
cat > .env << 'EOF'
OLLAMA_BASE_URL=http://localhost:11434/api
OLLAMA_MODEL=gemma4:e4b
EOF
```

:::message
**他のモデルに切り替えたい場合**
`OLLAMA_MODEL` の値を変えるだけで切り替えられます（`gemma4:e2b` / `qwen3.6:27b` など）。再起動 (`npm run dev` の再実行) を忘れずに。
:::

---

## Step 4: Mastra エージェントを定義する

エージェントが使う Ollama モデル設定を作ります。

```bash
mkdir -p src/mastra/agents
```

**`src/mastra/agents/chat-agent.ts`** を作成します。

```typescript
import { Agent } from "@mastra/core/agent";
import { createOllama } from "ollama-ai-provider-v2";

const ollama = createOllama({
  baseURL: process.env.OLLAMA_BASE_URL || "http://localhost:11434/api",
});

export const chatAgent = new Agent({
  id: "chatAgent",
  name: "Local Chat Agent",
  instructions: `あなたはローカルで動作する日本語アシスタントです。
- 回答は簡潔に、必要なら箇条書きで。
- 不確かなことは「分かりません」と素直に答えます。
- コードは TypeScript を優先します。`,
  model: ollama(process.env.OLLAMA_MODEL || "gemma4:e4b"),
});
```

:::message
**`id` フィールドは必須です。**
`@mastra/core@1.x` 系では Agent コンストラクタに `id` を渡さないと型エラーになります。`id` の値が後述の API ルートで指定する `agentId` と一致するようにします。
:::

**`src/mastra/index.ts`** を作成します。

```typescript
import { Mastra } from "@mastra/core";
import { chatAgent } from "./agents/chat-agent";

export const mastra = new Mastra({
  agents: { chatAgent },
});
```

---

## Step 5: API ルートを作る

Next.js の API ルートで、Mastra エージェントのストリーミング応答を AI SDK の形式に変換します。

```bash
mkdir -p src/app/api/chat
```

**`src/app/api/chat/route.ts`** を作成します。

```typescript
import { handleChatStream } from "@mastra/ai-sdk";
import { createUIMessageStreamResponse } from "ai";
import { mastra } from "@/mastra";

export async function POST(req: Request) {
  const params = await req.json();

  const stream = await handleChatStream({
    mastra,
    agentId: "chatAgent",
    params,
  });

  return createUIMessageStreamResponse({ stream });
}
```

---

## Step 6: チャット画面を作る

**`src/app/chat/page.tsx`** を作成します。

```bash
mkdir -p src/app/chat
```

```tsx
"use client";

import { useState } from "react";
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

import {
  Conversation,
  ConversationContent,
  ConversationScrollButton,
} from "@/components/ai-elements/conversation";
import {
  Message,
  MessageContent,
  MessageResponse,
} from "@/components/ai-elements/message";
import {
  PromptInput,
  PromptInputBody,
  PromptInputTextarea,
} from "@/components/ai-elements/prompt-input";

export default function ChatPage() {
  const [input, setInput] = useState("");

  const { messages, sendMessage, status } = useChat({
    transport: new DefaultChatTransport({ api: "/api/chat" }),
  });

  const handleSubmit = () => {
    if (!input.trim()) return;
    sendMessage({ text: input });
    setInput("");
  };

  return (
    <div className="h-screen w-full p-6">
      <div className="flex h-full flex-col">
        <Conversation className="flex-1">
          <ConversationContent>
            {messages.map((message) => (
              <div key={message.id}>
                {message.parts?.map((part, i) => {
                  if (part.type === "text") {
                    return (
                      <Message
                        key={`${message.id}-${i}`}
                        from={message.role}
                      >
                        <MessageContent>
                          <MessageResponse>{part.text}</MessageResponse>
                        </MessageContent>
                      </Message>
                    );
                  }
                  return null;
                })}
              </div>
            ))}
            <ConversationScrollButton />
          </ConversationContent>
        </Conversation>

        <PromptInput onSubmit={handleSubmit} className="mt-6">
          <PromptInputBody>
            <PromptInputTextarea
              value={input}
              onChange={(e) => setInput(e.target.value)}
              placeholder="メッセージを入力（Enter で送信）"
              disabled={status !== "ready"}
            />
          </PromptInputBody>
        </PromptInput>
      </div>
    </div>
  );
}
```

---

## Step 7: 起動して動作確認する

開発サーバーを起動します。

```bash
npm run dev
```

ブラウザで [http://localhost:3000/chat](http://localhost:3000/chat) を開き、メッセージを送信します。

**完了条件**

- [ ] 入力したメッセージがすぐに送信欄から消える
- [ ] アシスタントの応答が**ストリーミング**で（少しずつ）表示される
- [ ] `ollama ps` を別ターミナルで実行すると、`OLLAMA_MODEL` で指定したモデルが起動している

```bash
ollama ps
```

---

## 発展: ツールを使えるエージェントにする

`qwen3.6:27b` や `gemma4` 系はツール呼び出し（function calling）に対応しています。
エージェントに「現在時刻を返すツール」を追加してみましょう。

**`src/mastra/tools/now-tool.ts`** を作成します。

```typescript
import { createTool } from "@mastra/core/tools";
import { z } from "zod";

export const nowTool = createTool({
  id: "now",
  description: "現在の日時を ISO 8601 形式で返します。",
  inputSchema: z.object({}),
  outputSchema: z.object({ now: z.string() }),
  execute: async () => ({ now: new Date().toISOString() }),
});
```

**`src/mastra/agents/chat-agent.ts`** を更新してツールを登録します。

```typescript
import { Agent } from "@mastra/core/agent";
import { createOllama } from "ollama-ai-provider-v2";
import { nowTool } from "../tools/now-tool";

const ollama = createOllama({
  baseURL: process.env.OLLAMA_BASE_URL || "http://localhost:11434/api",
});

export const chatAgent = new Agent({
  id: "chatAgent",
  name: "Local Chat Agent",
  instructions: `あなたはローカルで動作する日本語アシスタントです。
時刻を聞かれたときは now ツールを呼び出して回答に含めます。`,
  model: ollama(process.env.OLLAMA_MODEL || "qwen3.6:27b"),
  tools: { nowTool },
});
```

ブラウザで「今何時？」と聞いてみてください。エージェントが `now` ツールを呼び出し、その結果を使って回答します。

:::message
ツール呼び出しは大型モデルのほうが安定します。`gemma4:e4b` でも動きますが、`qwen3.6:27b` のほうが指示への追従が良いことが多いです。
:::

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| `ECONNREFUSED 127.0.0.1:11434` | Ollama が起動していない。`curl http://localhost:11434` で確認 |
| `model not found` | `OLLAMA_MODEL` の名前が `ollama list` に存在するか確認 |
| `Property 'id' is missing in type` | Agent に `id: "chatAgent"` が抜けている。Step 4 のコードと一致させる |
| `npm run build` が ai-elements の型エラーで落ちる | `npx ai-elements@latest` を引数なしで実行した可能性。`rm -rf src/components/ai-elements src/components/ui` してから `npx ai-elements@latest add conversation message prompt-input` を再実行 |
| 応答が極端に遅い | 別の大型モデルが常駐していないか `ollama ps` で確認、不要なら `ollama stop` |
| ストリーミングされず一括で出る | `ai` `@ai-sdk/react` `ollama-ai-provider-v2` のバージョン整合を確認。それぞれ `^5` `^2` `^1.5.5` であること |
| `Cannot find module '@/mastra'` | `tsconfig.json` の `paths` に `"@/*": ["./src/*"]` が入っているか確認（`--src-dir` で作成していれば既定で入る） |

---

## チェックリスト

- [ ] `npm run dev` で `http://localhost:3000/chat` が開ける
- [ ] チャット画面でローカル Ollama モデルからストリーミング応答が返る
- [ ] `OLLAMA_MODEL` を変えるとモデルが切り替わる
- [ ] （任意）`now` ツールを追加してツール呼び出しが動作した

確認できたら、次章（トラブルシューティング）に進みます。
