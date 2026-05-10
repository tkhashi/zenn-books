---
title: "トラブルシューティング"
free: true
---

## よく使う確認コマンド

```bash
ollama list           # ダウンロード済みモデル一覧
ollama ps             # 起動中のモデル
ollama stop <model>   # モデルを停止
```

---

## 症状別の対処法

| 症状 | 原因候補 | 対処 |
|---|---|---|
| `model requires more system memory` | モデルが大きい・context が長い | 小さいモデルへ変更。Continue の `contextLength` を `4096` か `2048` に下げる |
| 応答が極端に遅い | 大型モデル・他アプリのメモリ消費 | 不要アプリを閉じる。`ollama ps` で常駐モデルを確認して停止する |
| 画像を読んでくれない | vision 非対応・パス誤り | `gemma4:e4b` か `qwen3.6:27b` を使う。画像パスを絶対パスにする |
| `404 model not found` | pull していない・タグ名違い | `ollama list` で名前確認。必要なら `ollama pull <モデル名>` |
| pull が遅い・止まる | 回線が遅い・モデルが大きい | 小さいモデルに切り替える。`qwen3.6:27b` はスキップしてよい |
| ストレージ不足 | モデルファイルが大きい | `ollama rm <モデル名>` で不要モデルを削除する |
| Continue で画像・tool が使えない | capability の自動検出失敗 | `config.yaml` に `image_input` / `tool_use` を明示する |
| `ollama run` が固まって応答しない | メモリ不足・モデルが起動中 | `Ctrl + C` で中断。`ollama stop <model>` で停止してから再試行 |

---

## Continue が Ollama に接続できない場合

1. Ollama が起動しているか確認する

```bash
curl http://localhost:11434
```

`Ollama is running` が返れば起動中です。返らない場合は Ollama を起動します。

2. `config.yaml` の `apiBase` が正しいか確認する

```yaml
apiBase: http://localhost:11434
```

3. VS Code を再起動する

---

## モデル名のスペルミスを確認する

```bash
ollama list
```

表示されるモデル名と `config.yaml` や `ollama run` で指定しているモデル名が一致しているか確認します。

よくある間違いです。

| 誤り | 正しい |
|---|---|
| `gemma4-e4b` | `gemma4:e4b` |
| `gemma:4e4b` | `gemma4:e4b` |
| `qwen3.6-27b` | `qwen3.6:27b` |
