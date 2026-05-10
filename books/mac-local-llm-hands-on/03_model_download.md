---
title: "モデルのダウンロード"
---

## この章でやること

- 自分のコースに合ったモデルをダウンロードする
- `ollama list` でダウンロード完了を確認する

---

:::message
**コース確認**
Ch.01 で決めたコース（Lite / Standard / Pro）に対応するセクションだけ実行してください。
他のコースのコマンドは実行しないでください。
:::

---

## Lite コースの方

**対象: Unified Memory 16 GB**

```bash
ollama pull gemma4:e4b
```

ダウンロードサイズは約 9.6 GB です。

:::message
メモリが 16 GB でモデル起動時に `model requires more system memory` と表示された場合は、より小さいモデルを使います。

```bash
ollama pull gemma4:e2b
```

`gemma4:e2b` は約 7.2 GB です。以降の章でモデル名として `gemma4:e4b` が出てきたら、`gemma4:e2b` に読み替えてください。
:::

---

## Standard コースの方

**対象: Unified Memory 24 GB / 32 GB**

```bash
ollama pull gemma4:e4b
```

ダウンロードサイズは約 9.6 GB です。

**32 GB かつ空き容量が 70 GB 以上ある場合のみ**、追加で以下も実行できます。

```bash
ollama pull qwen3.6:27b
```

ダウンロードサイズは約 17 GB です。

:::message alert
`gemma4:e4b` と `qwen3.6:27b` を**同時に動かさないでください**。
使い終わったら `ollama stop <モデル名>` で停止してから、次のモデルを起動します。
:::

---

## Pro コースの方

**対象: Unified Memory 48 GB 以上**

```bash
ollama pull qwen3.6:27b
ollama pull gemma4:26b
```

ダウンロードサイズの目安です。

| モデル | サイズ |
|---|---|
| `qwen3.6:27b` | 約 17 GB |
| `gemma4:26b` | 約 18 GB |

**Unified Memory が 64 GB 以上の場合**は、`gemma4:26b` の代わりに `:31b` も選択できます。

```bash
ollama pull gemma4:31b
```

| モデル | サイズ |
|---|---|
| `gemma4:31b` | 約 20 GB |

:::message
48 GB Mac では `gemma4:26b` と `qwen3.6:27b` を同時に動かせますが、動作が重いと感じたら片方を `ollama stop` で停止してください。
:::

---

## ダウンロード完了の確認

すべてのダウンロードが終わったら確認します。

```bash
ollama list
```

出力例（Standard コースで両モデルをダウンロードした場合）：

```text
NAME              ID              SIZE      MODIFIED
gemma4:e4b        xxxxxxxx        9.6 GB    ...
qwen3.6:27b       xxxxxxxx        17 GB     ...
```

自分がダウンロードしたモデルが一覧に表示されていれば OK です。

---

## チェックリスト

- [ ] 自分のコースに対応するモデルをダウンロードした
- [ ] `ollama list` にダウンロードしたモデルが表示された

確認できたら、次章（マルチモーダルチャット）に進みます。
