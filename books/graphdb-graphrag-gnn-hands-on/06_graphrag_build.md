---
title: "GraphRAGを作る — LlamaIndex でナレッジグラフを構築する"
free: true
---

## この章でやること

- LlamaIndex を使って映画概要テキストを読み込む
- LLM（gpt-4o-mini）でエンティティと関係を抽出する
- 抽出結果を Neo4j にグラフとして書き込む

この章では OpenAI API を使用します。**推定コスト: 約 $0.02〜0.05**（映画50件の場合）。

:::message alert
**OpenAI 新規アカウントは無料枠が無くなりました**。`insufficient_quota` (HTTP 429) エラーが出る場合は、[OpenAI 課金ページ](https://platform.openai.com/settings/organization/billing) で最低 $5 のクレジットを購入してから再実行してください。
:::

## 使うファイル

```
scripts/build_graph_rag.py
data/movies_overview_sample.csv
```

## コードの全体像

`build_graph_rag.py` の処理の流れ：

```python
# 1. 設定読み込み
load_dotenv()
openai_api_key = os.getenv("OPENAI_API_KEY")

# 2. LLM と Embedding モデルの設定（リトライ込み）
llm = OpenAI(model="gpt-4o-mini", api_key=openai_api_key, max_retries=5)
embed_model = OpenAIEmbedding(model="text-embedding-3-small", api_key=openai_api_key, max_retries=5)

# 3. Neo4j への接続
graph_store = Neo4jPropertyGraphStore(
    url="bolt://localhost:7687",
    username="neo4j",
    password="password123",
)

# 4. ドキュメント読み込み
documents = load_movie_documents("data/movies_overview_sample.csv")

# 5. グラフインデックス構築（LLM がエンティティ・関係を抽出）
index = PropertyGraphIndex.from_documents(
    documents,
    llm=llm,
    embed_model=embed_model,
    property_graph_store=graph_store,
    show_progress=True,
)

# 6. インデックスを保存
index.storage_context.persist(persist_dir="./storage")
```

ポイントは `PropertyGraphIndex.from_documents()` です。LlamaIndex が各ドキュメントを LLM に渡し、エンティティ（映画・人物・場所）と関係（監督した・出演した）を自動抽出して Neo4j に書き込みます。

## Step 1: グラフを構築する

```bash
uv run python scripts/build_graph_rag.py
```

実行中は映画1件ずつ LLM に処理を投げるため、進捗バーが表示されます。

```
Loading data/movies_overview_sample.csv ...
  51 documents loaded.
Building PropertyGraphIndex (this calls OpenAI; takes a few minutes)...
Applying transformations: 100%|██████████| 2/2 [00:25<00:00]
Generating embeddings: 100%|██████████| 9/9 [00:01<00:00]
Index persisted to ./storage
Neo4j total: 3182 nodes, 103872 relationships.
GraphRAG-extracted entities: 436
```

LLM 抽出は確率的なため、エンティティ数は実行ごとに変わります。映画50件で **400〜600 のエンティティ**が抽出されれば正常です。`Neo4j total` には Ch.03 でインポートした User/Movie/Genre ノード（約 2644 件）が含まれているため、純粋な GraphRAG エンティティは `__Entity__` ラベルでフィルタした 436 件です。

所要時間の目安: **30秒〜1分**（映画50件、gpt-4o-mini）

:::message
**APIレート制限エラーが出た場合**

LLM クライアントに `max_retries=5` を設定済みなので、通常は自動でリトライされます。それでも `RateLimitError` が解消しない場合は OpenAI ダッシュボードでアカウントの Tier を確認してください（新規アカウントは Tier 1 で TPD 上限が低めです）。
:::

## Step 2: Neo4j Browser で確認

グラフが書き込まれたか確認します。`http://localhost:7474` を開き、以下を実行：

```cypher
MATCH (n)-[r]->(m)
WHERE NOT n:Movie OR NOT m:Movie
RETURN n, r, m
LIMIT 50
```

LLM が抽出したエンティティ（人物・場所・概念）とその関係が可視化されます。

**抽出されたエンティティ数を確認**：

```cypher
MATCH (n:__Entity__)
RETURN count(n) AS count
```

LlamaIndex の PropertyGraphIndex は抽出したエンティティすべてに `__Entity__` ラベルを付与します。映画50件で約 400〜600 件のエンティティになります。

**サンプルエンティティを見る**：

```cypher
MATCH (n:__Entity__)
RETURN n.name LIMIT 10
```

```
Toy story
Movie
1995
Woody
Cowboy doll
Buzz lightyear
Spaceman action figure
Andy
Young
John lasseter
```

人物（Woody, Andy, John lasseter）、年（1995）、概念（Movie, Cowboy doll）など多様なエンティティが抽出されます。

## 何が抽出されるか（実例）

映画「Toy Story (1995)」の概要から、実際に以下のような関係が抽出されます。

```cypher
MATCH (n:__Entity__ {name: "Toy story"})-[r]-(m)
RETURN type(r) AS rel, m.name AS name LIMIT 10
```

```
Directed     -> John lasseter
Features     -> Woody and buzz lightyear
Released in  -> 1995
Is           -> Movie
```

LLM がテキストを読み取り、エンティティ（Toy story, John lasseter, Woody）と関係（Directed, Features, Released in）を抽出して Neo4j に書き込んだ結果です。

:::message
**抽出結果は LLM の判断次第で揺らぎます**。たとえば「Star Wars (1977)」の概要から `George Lucas` という人物エンティティが抽出されないことがあります（LLM が「監督名」を抽出すべきと判断しないケース）。次章で、この特性が回答品質にどう影響するかを確認します。
:::

## コスト確認

```bash
uv run python scripts/check_openai_usage.py
```

実際のトークン消費量と費用が確認できます。

## 動作確認チェックリスト

- [ ] `build_graph_rag.py` がエラーなく完走した
- [ ] Neo4j Browser で Person や Place のノードが確認できる
- [ ] `./storage/` ディレクトリが作成されている

次章では、このグラフに質問して回答を得ます。
