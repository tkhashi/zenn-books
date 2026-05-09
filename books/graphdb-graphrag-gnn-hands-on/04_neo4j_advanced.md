---
title: "Neo4jを深める — インデックス・集計・パスクエリ"
free: true
---

## この章でやること

- インデックスを作成してクエリを速くする
- 集計クエリで映画ランキングを作る
- グラフを辿って「2ホップ推薦」を実装する

## Step 1: インデックスを作成する

インデックスを作ると、`WHERE` 句の絞り込みが高速になります。

```cypher
CREATE INDEX movie_title IF NOT EXISTS FOR (m:Movie) ON (m.title);
CREATE INDEX user_id IF NOT EXISTS FOR (u:User) ON (u.id);
```

作成確認：

```cypher
SHOW INDEXES
```

`movie_title`、`user_id` が `ONLINE` 状態になっていれば OK です。

## 集計クエリ

### 人気映画ランキング（評価数順）

```cypher
MATCH (u:User)-[r:RATED]->(m:Movie)
RETURN m.title, count(r) AS num_ratings, round(avg(r.rating), 2) AS avg_rating
ORDER BY num_ratings DESC
LIMIT 10
```

`count(r)` でエッジの数を、`avg(r.rating)` でプロパティの平均を計算します。

### ジャンル別の平均評価

```cypher
MATCH (m:Movie)-[:BELONGS_TO]->(g:Genre)
MATCH (u:User)-[r:RATED]->(m)
RETURN g.name, round(avg(r.rating), 2) AS avg_rating, count(r) AS total_ratings
ORDER BY avg_rating DESC
```

### 特定ユーザーの評価傾向

```cypher
MATCH (u:User {id: 196})-[r:RATED]->(m:Movie)-[:BELONGS_TO]->(g:Genre)
RETURN g.name, count(r) AS rated_count, round(avg(r.rating), 2) AS avg_rating
ORDER BY avg_rating DESC
```

ユーザー 196 がどのジャンルを高く評価しているかがわかります。

## 2ホップ推薦：グラフを辿って映画を推薦する

グラフDB の真骨頂は「関係を辿る」ことです。

「自分と同じ映画を評価した他ユーザーが、自分はまだ見ていない映画を評価していたら推薦する」というロジックを Cypher で書けます。

```cypher
MATCH (me:User {id: 196})-[r1:RATED]->(m:Movie)<-[r2:RATED]-(other:User)
MATCH (other)-[r3:RATED]->(rec:Movie)
WHERE r1.rating >= 4
  AND r2.rating >= 4
  AND r3.rating >= 4
  AND NOT (me)-[:RATED]->(rec)
RETURN rec.title, count(other) AS recommenders
ORDER BY recommenders DESC
LIMIT 10
```

クエリの読み方：

1. `me` が高評価（4以上）した映画 `m` を取得
2. 同じ映画 `m` を高評価した別ユーザー `other` を取得
3. `other` が高評価した映画 `rec` のうち、`me` がまだ見ていないものを取得
4. 推薦してくれる人数（`recommenders`）が多い順に並べる

```
Star Wars (1977)                        12
Schindler's List (1993)                 10
Shawshank Redemption, The (1994)         9
...
```

RDB でこれを書くと複数の JOIN が必要ですが、グラフなら構造を直接辿るだけです。

## クエリプランの確認

重いクエリの原因を調べるには `EXPLAIN` を使います。

```cypher
EXPLAIN
MATCH (u:User {id: 196})-[r:RATED]->(m:Movie)
RETURN m.title, r.rating
```

`NodeIndexSeek` が表示されていれば、インデックスが効いています。`AllNodesScan` が表示されている場合は全件スキャンになっているため、インデックスを追加することを検討します。

## Python でのバッチクエリ

複数クエリを一括で実行する場合は `session.execute_write()` を使います。

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password123"))

def get_recommendations(tx, user_id: str, limit: int = 10):
    result = tx.run("""
        MATCH (me:User {id: $user_id})-[r1:RATED]->(m:Movie)<-[r2:RATED]-(other:User)
        MATCH (other)-[r3:RATED]->(rec:Movie)
        WHERE r1.rating >= 4 AND r2.rating >= 4 AND r3.rating >= 4
          AND NOT (me)-[:RATED]->(rec)
        RETURN rec.title AS title, count(other) AS score
        ORDER BY score DESC
        LIMIT $limit
    """, user_id=user_id, limit=limit)
    return [record["title"] for record in result]

with driver.session() as session:
    recs = session.execute_read(get_recommendations, user_id="196")
    for title in recs:
        print(title)

driver.close()
```

実行：

```bash
uv run python scripts/recommend_cypher.py
```

## 動作確認チェックリスト

- [ ] `SHOW INDEXES` でインデックスが `ONLINE` になっている
- [ ] 2ホップ推薦クエリが結果を返す
- [ ] `recommend_cypher.py` が推薦映画を10件出力する

次章からは GraphRAG に進みます。これまで「構造化されたデータ」をグラフで扱いましたが、次は「テキスト」をグラフ化します。
