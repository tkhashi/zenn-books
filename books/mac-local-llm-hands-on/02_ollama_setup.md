---
title: "Ollama のインストール"
free: true
---

## この章でやること

- Ollama をインストールする
- Ollama が正常に起動していることを確認する

---

## Ollama とは

Ollama は、ローカルで大規模言語モデルを実行するためのランタイムです。
モデルのダウンロード・起動・停止・削除を `ollama` コマンド1つで管理できます。

Apple Silicon Mac では CPU と GPU を統合したユニファイドメモリを活用するため、推論が高速です。

---

## インストール方法

**2通りの方法があります。どちらか一方だけ実行してください。**

### 方法A: 公式アプリ（推奨）

1. [Ollama 公式サイト](https://ollama.com) から macOS 版をダウンロードする
2. ダウンロードした `.zip` を展開して `Ollama.app` を `/Applications` に移動する
3. `Ollama.app` を起動する
4. 初回起動時にインストール確認が出たら「Install」を選ぶ

メニューバーにラマのアイコンが表示されれば起動完了です。

### 方法B: Homebrew

Homebrew を使っている場合はこちらが便利です。

```bash
brew install ollama
```

インストール後、以下でサービスを起動します。

```bash
ollama serve
```

:::message
`ollama serve` はターミナルを閉じると停止します。
常時起動させたい場合は、別ターミナルを使うか `brew services start ollama` で常駐化できます。
:::

---

## 動作確認

どちらの方法でインストールしても、以下のコマンドで確認します。

```bash
ollama --version
```

バージョン番号が表示されれば OK です。

続けて、API が応答しているか確認します。

```bash
curl http://localhost:11434
```

`Ollama is running` と返ってくれば正常です。

---

## チェックリスト

- [ ] `ollama --version` でバージョンが表示された
- [ ] `curl http://localhost:11434` で `Ollama is running` が返った

確認できたら、次章（モデルのダウンロード）に進みます。
