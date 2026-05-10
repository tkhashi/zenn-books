---
title: "後片付け — モデルの削除とストレージ回復"
free: true
---

## この章でやること

- ダウンロードしたモデルを削除してストレージを回復する
- 必要に応じて Ollama 本体と Continue 拡張を削除する

:::message alert
**この章は必ず実施してください。**
モデルファイルは 7 GB〜20 GB のストレージを占有します。ハンズオン終了後は削除して回収しましょう。
:::

---

## Step 1: 現在の状況を確認する

```bash
df -h ~
du -sh ~/.ollama/models 2>/dev/null || true
ollama list
ollama ps
```

---

## Step 2: 起動中のモデルを停止する

`ollama ps` に表示されているモデルをすべて停止します。

```bash
ollama stop gemma4:e2b 2>/dev/null || true
ollama stop gemma4:e4b 2>/dev/null || true
ollama stop qwen3.6:27b 2>/dev/null || true
ollama stop gemma4:26b 2>/dev/null || true
ollama stop gemma4:31b 2>/dev/null || true
```

---

## Step 3: モデルを削除する

ダウンロードしたモデルをまとめて削除します。

```bash
for model in gemma4:e2b gemma4:e4b qwen3.6:27b gemma4:26b gemma4:31b; do
  ollama rm "$model" 2>/dev/null || true
done
```

削除後に確認します。

```bash
ollama list
du -sh ~/.ollama/models 2>/dev/null || true
df -h ~
```

`ollama list` が空になり、ストレージが回復していれば OK です。

---

## Step 4: サンプルコードを削除する（任意）

```bash
rm -rf ~/local-llm-hands-on
```

---

## Step 5: Ollama 本体を削除する（任意）

今後もローカル LLM を使うなら Ollama は残して構いません。
完全に削除したい場合のみ実行します。

### 公式アプリで入れた場合

1. メニューバーの Ollama を右クリックして「Quit Ollama」
2. `/Applications/Ollama.app` をゴミ箱に移動
3. 以下でデータディレクトリを削除（任意）

```bash
rm -rf ~/.ollama
sudo rm -f /usr/local/bin/ollama
```

### Homebrew で入れた場合

```bash
brew uninstall ollama
rm -rf ~/.ollama
```

---

## Step 6: Continue 拡張を削除する（任意）

VS Code の Extensions から Continue をアンインストールします。
CLI で削除する場合は以下です。

```bash
code --uninstall-extension Continue.continue
```

---

## 参考リンク

確認日: 2026-05-10

| リンク | 内容 |
|---|---|
| [Gemma 4 model card](https://ai.google.dev/gemma/docs/core/model_card_4?hl=ja) | Gemma 4 の公式モデルカード |
| [Qwen3.6-27B official blog](https://qwen.ai/blog?id=qwen3.6-27b) | Qwen3.6 の公式ブログ |
| [Ollama Gemma 4 library](https://www.ollama.com/library/gemma4) | Ollama の Gemma 4 ページ |
| [Ollama Qwen3.6 library](https://ollama.com/library/qwen3.6) | Ollama の Qwen3.6 ページ |
| [Ollama macOS docs](https://docs.ollama.com/macos) | Ollama macOS ドキュメント |
| [Ollama Vision docs](https://docs.ollama.com/capabilities/vision) | Ollama Vision ドキュメント |
| [Continue Ollama provider docs](https://docs.continue.dev/customize/model-providers/top-level/ollama) | Continue の Ollama 設定ガイド |

---

## チェックリスト

- [ ] `ollama list` が空になった
- [ ] ストレージが回復した
- [ ] サンプルコードを削除した（任意）
- [ ] Ollama 本体を削除した（任意）

お疲れ様でした。
