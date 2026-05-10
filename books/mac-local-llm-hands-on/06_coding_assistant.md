---
title: "VS Code + Continue でコーディング支援"
---

## この章でやること

- Continue 拡張をインストールする
- Ollama と接続する設定を行う
- サンプルコードを用意する

---

:::message
**コース確認**
重いと感じる場合は `gemma4:e4b` だけで進めてください。
大型モデルを使う場合は、他のモデルが起動中でないことを確認してから行います。

```bash
ollama ps
ollama stop <起動中のモデル名>
```
:::

---

## Step 1: Continue をインストールする

1. VS Code を開く
2. Extensions（`Cmd + Shift + X`）で `Continue` を検索する
3. **Continue** をインストールする
4. Ollama が起動していることを確認する

```bash
curl http://localhost:11434
ollama list
```

---

## Step 2: Continue の設定ファイルを書く

`~/.continue/config.yaml` に以下を設定します。

**ダウンロードしていないモデルのブロックは削除してください。**

```yaml
name: Local LLM Hands-on
version: 0.0.1
schema: v1

models:
  - name: Gemma 4 E4B
    provider: ollama
    model: gemma4:e4b
    apiBase: http://localhost:11434
    roles:
      - chat
      - edit
    capabilities:
      - image_input
    defaultCompletionOptions:
      contextLength: 8192
      temperature: 0.2
      numPredict: 2048

  - name: Qwen3.6 27B
    provider: ollama
    model: qwen3.6:27b
    apiBase: http://localhost:11434
    roles:
      - chat
      - edit
      - apply
    capabilities:
      - image_input
      - tool_use
    defaultCompletionOptions:
      contextLength: 8192
      temperature: 0.2
      numPredict: 2048
```

:::message
**`model requires more system memory` が出た場合**
`contextLength` を下げます。

```yaml
defaultCompletionOptions:
  contextLength: 4096
  temperature: 0.2
  numPredict: 1024
```

それでも重い場合は `contextLength: 2048` まで下げてください。
:::

:::message
**コース別の設定方針**

| コース | 設定するモデル |
|---|---|
| Lite | `Gemma 4 E4B` のみ（`e2b` に変更した方はモデル名も変える） |
| Standard（e4b のみ） | `Gemma 4 E4B` のみ |
| Standard（qwen3.6 あり） | 両方設定。同時起動しないよう注意 |
| Pro | `Qwen3.6 27B` + Pro モデルを追加 |
:::

---

## Step 3: サンプルコードを用意する

演習用の TypeScript ファイルを作ります。

```bash
mkdir -p ~/local-llm-hands-on/src
```

以下のコードを `~/local-llm-hands-on/src/todo.ts` として保存します。

```typescript
export type Todo = {
  id: string;
  title: string;
  done: boolean;
  createdAt: string;
};

export function addTodo(todos: Todo[], title: string): Todo[] {
  if (!title) {
    return todos;
  }

  return [
    ...todos,
    {
      id: String(Date.now()),
      title,
      done: false,
      createdAt: new Date().toISOString(),
    },
  ];
}

export function toggleTodo(todos: Todo[], id: string): Todo[] {
  return todos.map((todo) => {
    if (todo.id === id) {
      return { ...todo, done: !todo.done };
    }
    return todo;
  });
}

export function filterTodos(todos: Todo[], filter: string): Todo[] {
  if (filter === "active") {
    return todos.filter((todo) => !todo.done);
  }
  if (filter === "done") {
    return todos.filter((todo) => todo.done);
  }
  return todos;
}
```

VS Code でディレクトリを開きます。

```bash
code ~/local-llm-hands-on
```

---

## チェックリスト

- [ ] Continue 拡張をインストールした
- [ ] `~/.continue/config.yaml` を設定した
- [ ] Continue のチャットパネルでモデルが選べる
- [ ] `~/local-llm-hands-on/src/todo.ts` を作成した

確認できたら、次章（コーディング演習）に進みます。
