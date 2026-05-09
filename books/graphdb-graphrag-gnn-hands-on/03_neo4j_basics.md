---
title: "Neo4j入門 — Cypherクエリで映画データを探索する"
free: true
---

## この章でやること

- MovieLens データを Neo4j にインポートする
- Cypher クエリの基本を学ぶ
- Python から Neo4j を操作する

## 使うファイル

```
scripts/import_movies.py   # データインポートスクリプト
```

## MovieLens データの構造

MovieLens 100K には3種類のデータがあります。

| ファイル | 内容 |
|---------|------|
| `u.data` | ユーザーID・映画ID・評価（1〜5）・タイムスタンプ |
| `u.item` | 映画ID・タイトル・公開年・ジャンル |
| `u.user` | ユーザーID・年齢・性別・職業 |

これをグラフ構造に変換すると以下のようになります。

```
(User {id: 196})-[:RATED {rating: 5}]->(Movie {id: 242})
(Movie)-[:BELONGS_TO]->(Genre)
```

:::message
本書では `User.id` と `Movie.id` を**整数**として格納します（MovieLens 元データが整数のため）。Cypher で指定する際は `{id: 196}` のように引用符なしで書きます。
:::

## Step 1: データを Neo4j にインポートする

```bash
uv run python scripts/import_movies.py
```

完了すると以下のようなメッセージが出ます。

```
Importing users... 943 users created.
Importing movies... 1682 movies created.
Importing genres... 19 genres created.
Importing ratings... 100000 ratings created.
Done.
```

所要時間の目安: **2〜3分**（M シリーズ Mac）

## Step 2: Neo4j Browser で確認

`http://localhost:7474` を開き、以下のクエリを実行します。

**全ノード数の確認**

```cypher
MATCH (n) RETURN labels(n) AS label, count(n) AS count
```

| label | count |
|-------|-------|
| Movie | 1682 |
| User | 943 |
| Genre | 19 |

**グラフを可視化する**

```cypher
MATCH (u:User)-[r:RATED]->(m:Movie)
RETURN u, r, m
LIMIT 20
```

ノードとエッジが可視化されたグラフが表示されます。

## Cypher の基本構文

Cypher は「見た目がグラフのまま」のクエリ言語です。

### ノードを検索する

```cypher
MATCH (m:Movie)
WHERE m.title = "Toy Story (1995)"
RETURN m
```

- `(m:Movie)` — Movie ラベルのノードを `m` として取得
- `WHERE` — 条件を絞り込む
- `RETURN` — 返す値を指定

### リレーションシップを辿る

```cypher
MATCH (m:Movie {title: "Toy Story (1995)"})-[:BELONGS_TO]->(g:Genre)
RETURN g.name
```

`-[:BELONGS_TO]->` がエッジの向きと種別を表します。

### 評価の高い映画を探す

```cypher
MATCH (u:User)-[r:RATED]->(m:Movie)
RETURN m.title, avg(r.rating) AS avg_rating, count(r) AS num_ratings
ORDER BY avg_rating DESC
LIMIT 10
```

### ユーザーが見ていない映画を探す

```cypher
MATCH (u:User {id: 196})
MATCH (m:Movie)
WHERE NOT (u)-[:RATED]->(m)
RETURN m.title
LIMIT 10
```

## Step 3: Python から操作する

Neo4j の Python ドライバを使うと、アプリケーションからクエリを実行できます。

```python
# scripts/query_example.py として実行
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password123")
)

with driver.session() as session:
    result = session.run(
        "MATCH (m:Movie) RETURN m.title AS title LIMIT 5"
    )
    for record in result:
        print(record["title"])

driver.close()
```

実行：

```bash
uv run python scripts/query_example.py
```

出力例：

```
Toy Story (1995)
GoldenEye (1995)
Four Rooms (1995)
Get Shorty (1995)
Copycat (1995)
```

## よくあるエラーと対処

**`ServiceUnavailable: Failed to establish connection`**

Neo4j が起動していない可能性があります。

```bash
docker compose ps          # STATUS が Up かを確認
docker compose logs neo4j  # エラーログを確認
docker compose up -d       # 起動していなければ起動
```

**`AuthError: The client is unauthorized`**

パスワードが違います。`password123` を確認してください。

## 動作確認チェックリスト

- [ ] `import_movies.py` が完走し、エラーが出ていない
- [ ] `MATCH (n) RETURN labels(n), count(n)` で Movie: 1682、User: 943、Genre: 19 が返る
- [ ] `query_example.py` が映画タイトルを5件出力する

次章では、より高度なクエリと推薦ロジックを学びます。
