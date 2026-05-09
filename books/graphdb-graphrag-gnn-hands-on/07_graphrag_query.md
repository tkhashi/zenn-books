---
title: "GraphRAGに質問する — コンテキスト付き回答の仕組み"
free: true
---

## この章でやること

- 構築したグラフに自然言語で質問する
- 回答がどうやって生成されるか仕組みを理解する
- 通常 RAG と回答品質を比較する

## 使うファイル

```
scripts/query_graph_rag.py
./storage/   (前章で作成したインデックス)
```

## Step 1: 質問スクリプトを実行する

```bash
uv run python scripts/query_graph_rag.py
```

対話形式で質問できます。

```
GraphRAG Query Engine ready.
Type your question (or 'quit' to exit):

> Toy Story に登場するキャラクターを教えてください
```

回答例（実測値、gpt-4o-mini）：

```
Toy Story に登場するキャラクターは、ウッディとバズ・ライトイヤーです。
```

短いですが、グラフから抽出した「Woody」「Buzz lightyear」というエンティティを正しく参照した回答です。データセットを大きくすればより詳細な回答（Andy、Sid 等）も得られます。

## 仕組みを理解する

クエリエンジンの内部で何が起きているか：

```
質問: "Toy Story に登場するキャラクター"
        ↓
1. [Embedding] 質問をベクトル化
        ↓
2. [グラフ検索] Neo4j で関連ノード・エッジを検索
   → "Toy Story" に繋がるエンティティを取得
   → Person, Character ノードを収集
        ↓
3. [コンテキスト構築] 取得したノード・エッジを文章化
   → "Woody is a character in Toy Story. Woody is_favorite_of Andy..."
        ↓
4. [LLM] コンテキストと質問を渡して回答生成
        ↓
回答
```

グラフを辿っているため、「キャラクター」という言葉がテキストになくても、`Person` や `Character` ノードに繋がるエッジから情報を取得できます。

## 通常 RAG と GraphRAG の比較

同じ質問を両方に投げて、回答の違いを確かめてみましょう。

```bash
# GraphRAG モード
echo "Toy Story の監督と Star Wars の監督を教えてください" | uv run python scripts/query_graph_rag.py
```

```
Toy Story の監督はジョン・ラセターです。
Star Wars の監督については情報が提供されていません。
```

```bash
# 通常 RAG（ベクトル検索）モード
echo "Toy Story の監督と Star Wars の監督を教えてください" | uv run python scripts/query_graph_rag.py --mode simple
```

```
Toy Story の監督はジョン・ラセターで、Star Wars の監督はジョージ・ルーカスです。
```

**興味深い結果**: 通常 RAG のほうが正しく答えています。理由は、CSV のテキストには「Directed by George Lucas」と書かれているのに、GraphRAG のエンティティ抽出時に LLM が `George Lucas` を `Star wars` のエンティティとして繋がなかったからです（前章末尾の `:::message` 参照）。

これは GraphRAG の「実装のクセ」を体感する重要な教訓です。

:::message
**GraphRAG が万能ではない**: グラフ抽出の精度はテキストの書き方・LLM のプロンプト・データセットの規模に強く依存します。実運用では:

- 大規模データ（数百〜数千ドキュメント）で効果が出やすい
- 抽出プロンプトをカスタマイズしてエンティティ・関係の取り漏らしを減らす
- GraphRAG と通常 RAG をハイブリッドで使う

などの工夫が必要です。
:::

## 使い分けの原則

| 方式 | 得意な質問 | 苦手な質問 |
|------|-----------|-----------|
| 通常 RAG | 1つのドキュメントで完結する質問 | 複数ドキュメントを跨ぐ関係の集約 |
| GraphRAG | エンティティ間の関係を辿る質問・全体像の俯瞰 | 抽出されなかった事実の検索 |

## 動作確認チェックリスト

- [ ] `query_graph_rag.py` が起動し、質問に回答できる
- [ ] Toy Story の登場キャラクターを答えられる
- [ ] `--mode simple` との回答の違いを確認できた

次章から GNN に進みます。GraphRAG は「テキストからグラフを作り LLM で回答」でしたが、GNN は「グラフ構造を機械学習して予測」します。
