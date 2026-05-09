---
title: "環境構築"
free: true
---

## この章でやること

- Docker Desktop の動作確認
- companion リポジトリのクローン
- Neo4j を Docker で起動
- Python 環境を uv で構築
- OpenAI API キーの設定
- データのダウンロード

## Step 1: Docker Desktop の確認

Docker Desktop が起動しているか確認します。

```bash
docker info
```

`Server: Docker Engine` と表示されれば OK です。`Cannot connect to the Docker daemon` と表示された場合は、Docker Desktop を起動してください（メニューバーのクジラアイコンをクリック）。

## Step 2: companion リポジトリのクローン

```bash
git clone https://github.com/tkhashi/graphdb-graphrag-gnn-handson
cd graphdb-graphrag-gnn-handson
```

以降の操作はすべてこのディレクトリ内で行います。

## Step 3: Neo4j を起動する

```bash
docker compose up -d
```

初回起動時は APOC プラグインのダウンロードが走るため、**起動完了まで60秒ほど**かかります。

起動確認：

```bash
docker compose ps
```

`neo4j-local` の `STATUS` が `Up` になっていれば OK です。

:::message
**8GB Mac の場合**: `docker-compose.yml` の以下3行を変更してから起動してください。

```yaml
NEO4J_server_memory_heap_initial__size: 256m
NEO4J_server_memory_heap_max__size: 512m
NEO4J_server_memory_pagecache_size: 256m
```

加えて Docker Desktop → Settings → Resources → Memory を最低 4GB に設定してください。
:::

## Step 4: Neo4j Browser で接続確認

ブラウザで `http://localhost:7474` を開きます。

ログイン情報：
- ユーザー名: `neo4j`
- パスワード: `password123`

ログイン後、上部の入力欄に以下を入力して実行してください（Ctrl+Enter または ▶ ボタン）。

```cypher
RETURN "Hello, Neo4j!" AS message
```

`"Hello, Neo4j!"` と返ってくれば接続成功です。

## Step 5: uv のインストール

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

インストール後、新しいターミナルを開くか、以下を実行して PATH を反映させます。

```bash
source ~/.zshrc
```

確認：

```bash
uv --version
# 例: uv 0.5.x (...)
```

## Step 6: Python 環境の構築

```bash
uv python install 3.11
uv python pin 3.11
uv sync
```

`uv sync` が終わると `.venv/` ディレクトリが作成されます。以降、スクリプトは `uv run python` で実行します。

インストール確認：

```bash
uv run python -c "import neo4j, torch, torch_geometric; from llama_index.core import Settings; from llama_index.graph_stores.neo4j import Neo4jPropertyGraphStore; print('OK')"
```

`OK` と表示されれば成功です。

:::message
**torch_geometric の import エラーが出た場合**
```bash
uv run python -c "import torch_geometric"
```
エラーが出たら `uv sync --reinstall` を試してください。
:::

## Step 7: OpenAI API キーの設定

companion リポジトリに `.env.example` があります。

```bash
cp .env.example .env
```

`.env` をテキストエディタで開き、`your-api-key-here` を実際のキーに書き換えます。

```
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxx
```

:::message alert
`.env` は `.gitignore` に登録済みです。**絶対にコミットしないでください。**
:::

API キーの疎通確認：

```bash
uv run python scripts/check_openai.py
```

`gpt-4o-mini is reachable` と表示されれば OK です。

## Step 8: データのダウンロード

```bash
uv run python scripts/download_data.py
```

以下のファイルが `data/` ディレクトリに展開されます。

| ファイル | 内容 | サイズ |
|---------|------|-------|
| `data/ml-100k/u.data` | ユーザーの映画評価（10万件） | 2MB |
| `data/ml-100k/u.item` | 映画メタデータ（1,682件） | 236KB |
| `data/ml-100k/u.user` | ユーザー属性（943件） | 22KB |
| `data/movies_overview_sample.csv` | 映画の概要テキスト（50件、GraphRAG用） | 45KB |

## 動作確認チェックリスト

この章が完了したら以下をすべて確認してください。

- [ ] `docker compose ps` で `neo4j-local` が `Up`
- [ ] `http://localhost:7474` でログインできる
- [ ] `uv run python -c "import neo4j, torch, torch_geometric; from llama_index.core import Settings; from llama_index.graph_stores.neo4j import Neo4jPropertyGraphStore; print('OK')"` が `OK` を表示
- [ ] `.env` に OpenAI API キーが設定されている
- [ ] `data/ml-100k/u.data` が存在する

すべて確認できたら、次章（Neo4j 入門）に進みましょう。
