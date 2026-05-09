---
title: "GNNを訓練・評価する — 損失関数・AUC・推薦精度"
free: true
---

## この章でやること

- GNN モデルを訓練する
- AUC で評価する
- 実際の推薦結果を出力する

## Step 1: 訓練を実行する

```bash
uv run python scripts/train_gnn.py
```

エポックごとに損失と AUC が表示されます（数値は実行ごとに変動します）。

```
Epoch 01 | loss=0.64 | val_auc=0.79
Epoch 02 | loss=0.55 | val_auc=0.83
Epoch 03 | loss=0.52 | val_auc=0.86
...
Epoch 10 | loss=0.42 | val_auc=0.90

Best val AUC: 0.90
Test AUC: 0.90
Saved best model to checkpoints/best_model.pt
```

所要時間の目安: **1〜2分**（Mac M シリーズ CPU、フルバッチ訓練）

:::message
**Val AUC が 0.80 以上** になれば、学習が正しく進んでいます。MovieLens 100K・10 エポックでは AUC 0.88〜0.90 が目安です（乱数初期化により数値はぶれます）。AUC が 0.70 未満で停滞する場合は、`T.ToUndirected()` 適用済みか・負例サンプリング（`sample_neg`）が動いているかを確認してください。
:::

## 損失関数の直感的な理解

訓練では **Binary Cross-Entropy（BCE）** を使います。

```python
import torch.nn.functional as F

# 正例エッジ（実際に評価した）のスコアは高く
# 負例エッジ（ランダムに選んだ未評価）のスコアは低く
loss = F.binary_cross_entropy_with_logits(pred_scores, edge_labels)
```

損失が下がる = 「実際に評価した映画のスコアが、ランダムな映画より高くなる」ように学習が進んでいる、ということです。

## AUC の直感的な理解

AUC（Area Under the ROC Curve）は「ランダムよりどれだけ良いか」を示します。

| AUC 値 | 意味 |
|--------|------|
| 0.5 | ランダム（学習できていない） |
| 0.65 | ある程度学習できている |
| 0.75 | 良好な推薦精度 |
| 1.0 | 完璧（過学習の疑いあり） |

## Step 2: 推薦結果を確認する

```bash
uv run python scripts/recommend_gnn.py --user_id 196
```

出力例（実際の映画タイトルは学習結果により変動します。スコアは sigmoid 通過後の 0〜1 確率）：

```
Recommendations for User 196:
  1. Star Wars (1977) [score: 0.80]
  2. Toy Story (1995) [score: 0.80]
  3. Return of the Jedi (1983) [score: 0.79]
  4. Contact (1997) [score: 0.79]
  5. Scream (1996) [score: 0.79]
  6. Full Monty, The (1997) [score: 0.78]
  7. Evita (1996) [score: 0.78]
  8. Air Force One (1997) [score: 0.78]
  9. Saint, The (1997) [score: 0.77]
 10. Liar Liar (1997) [score: 0.77]
```

このユーザーの過去の高評価ジャンルを考慮した推薦になっているか確認してみてください。

```bash
# ユーザー196 が高評価（4以上）した映画を確認
uv run python scripts/show_user_history.py --user_id 196 --min_rating 4
```

過去の高評価映画と推薦映画のジャンルが近ければ、GNN が傾向を捉えています。

## 別のユーザーで試す

任意のユーザー ID（1〜943）で試せます。

```bash
uv run python scripts/recommend_gnn.py --user_id 1
uv run python scripts/recommend_gnn.py --user_id 500
```

ユーザーによって推薦内容が変わることを確認してください。

## Cypher 推薦と GNN 推薦の比較

Ch.04 で実装した Cypher による2ホップ推薦と GNN 推薦を比べてみましょう。

```bash
uv run python scripts/compare_recommenders.py --user_id 196
```

```
=== Cypher (2-hop) Recommendations ===
  1. Star Wars (1977) [co-raters: 1503]
  2. Fargo (1996) [co-raters: 1340]
  3. Raiders of the Lost Ark (1981) [co-raters: 1254]
  4. Silence of the Lambs, The (1991) [co-raters: 1242]
  5. Return of the Jedi (1983) [co-raters: 1173]
  ...

=== GNN Recommendations ===
  1. Star Wars (1977) [score: 0.80]
  2. Toy Story (1995) [score: 0.80]
  3. Return of the Jedi (1983) [score: 0.79]
  4. Contact (1997) [score: 0.79]
  5. Scream (1996) [score: 0.79]
  ...
```

両者で **Star Wars / Return of the Jedi** など SF 大作が共通して上位に来ています。Cypher は「同じ映画を高評価したユーザーの数」というシンプルなルール、GNN は「グラフ構造から学習した埋め込み」と内部の仕組みは違いますが、ユーザー196 が SF を好む傾向を別ルートで捉えていることがわかります。

一方、GNN のほうが 1997 年公開の作品（Contact, Scream, Air Force One など）を多く挙げているのは、ユーザー196 の評価時期の傾向を埋め込みに反映しているためです。Cypher は co-rater の数だけを見るので、データセット全体で人気の古典的名作（Fargo, Godfather）が上位に出やすい傾向があります。

## 動作確認チェックリスト

- [ ] 10 エポック完走し `checkpoints/best_model.pt` が保存された
- [ ] Test AUC が 0.65 以上
- [ ] `recommend_gnn.py` が推薦映画 10 件を出力した
- [ ] ユーザーを変えると推薦内容が変わることを確認した

これで3つの技術をすべて体験しました。最終章で全体を振り返ります。
