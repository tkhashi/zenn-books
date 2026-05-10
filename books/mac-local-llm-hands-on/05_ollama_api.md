---
title: "Ollama API で画像チャットを試す"
---

## この章でやること

- Ollama の REST API を curl で呼び出す
- 画像をBase64エンコードしてAPIに渡す方法を理解する

---

## CLI との違い

前章では `ollama run` コマンドで直接チャットしました。
この章では Ollama の HTTP API（`localhost:11434`）を使います。

Web アプリやバックエンドからローカルLLMを使う際に必要な知識です。

---

## 画像をBase64エンコードして送信する

```bash
# 画像を Base64 エンコードして変数に格納
IMG=$(base64 < ~/Desktop/sample.png | tr -d '\n')

# API 呼び出し
curl -X POST http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemma4:e4b",
    "messages": [
      {
        "role": "user",
        "content": "この画像をUIレビューしてください。改善点を箇条書きで出してください。",
        "images": ["'"$IMG"'"]
      }
    ],
    "stream": false
  }'
```

:::message
**モデル名の変更方法**
`"model"` の値を自分のコースのモデル名に変えます。

| コース | model の値 |
|---|---|
| Lite | `"gemma4:e2b"` または `"gemma4:e4b"` |
| Standard | `"gemma4:e4b"` または `"qwen3.6:27b"` |
| Pro | `"qwen3.6:27b"` または `"gemma4:26b"` |
:::

---

## レスポンスの確認

API レスポンスは JSON で返ります。

```json
{
  "model": "gemma4:e4b",
  "message": {
    "role": "assistant",
    "content": "このUIの改善点は以下の通りです。..."
  },
  "done": true
}
```

`message.content` の中にモデルの回答が入っています。

---

## テキストのみの API 呼び出し

画像なしのテキストチャットは `images` フィールドを省略するだけです。

```bash
curl -X POST http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemma4:e4b",
    "messages": [
      {
        "role": "user",
        "content": "TypeScriptのinterfaceとtypeの違いを教えてください。"
      }
    ],
    "stream": false
  }'
```

---

## チェックリスト

- [ ] 画像を渡した API 呼び出しが成功した
- [ ] レスポンス JSON の `message.content` に回答が入っていた

確認できたら、次章（VS Code + Continue 設定）に進みます。
