---
title: "GNNを作る — PyTorch Geometric でリンク予測モデルを書く"
free: true
---

## この章でやること

- Neo4j のグラフを PyTorch Geometric のデータ形式に変換する
- GNN モデル（GraphSAGE）を定義する
- データローダーを設定する

## 使うファイル

```
scripts/prepare_pyg_data.py   # Neo4j → PyG データ変換
scripts/gnn_model.py          # モデル定義
```

## Step 1: Neo4j データを PyG 形式に変換する

```bash
uv run python scripts/prepare_pyg_data.py
```

処理が完了すると `data/pyg_data.pt` が作成されます。

```
Loading ratings from Neo4j...
Users: 943, Movies: 1682, Ratings: 100000
Building HeteroData...
Train edges: 80000, Val edges: 10000, Test edges: 10000
Saved to data/pyg_data.pt
```

### 変換の中身を理解する

`prepare_pyg_data.py` の核心部分：

```python
import torch
import torch_geometric.transforms as T
from torch_geometric.data import HeteroData

data = HeteroData()

# ユーザーは特徴量を持たないため、後段でEmbeddingレイヤーから学習する
# （Embeddingのため、ここでは num_nodes だけ宣言）
data["user"].num_nodes = 943

# 映画ノードの特徴量（ジャンルのマルチホットエンコーディング）
data["movie"].x = movie_features          # shape: [1682, 19]

# エッジ（評価関係）
data["user", "rates", "movie"].edge_index = edge_index   # shape: [2, 100000]
data["user", "rates", "movie"].edge_label = ratings      # shape: [100000]

# user→movie のみだとユーザー側に情報が伝播しないため、逆向きエッジを自動追加
data = T.ToUndirected()(data)
```

`HeteroData` では `data["node_type"]` と `data["src", "relation", "dst"]` でノードとエッジを管理します。`T.ToUndirected()` が `(movie, rev_rates, user)` という逆向きエッジを自動生成し、ユーザー側にも隣人情報が流れるようになります。

## Step 2: モデルを定義する

PyG では **同種グラフ用のモデルを書き、`to_hetero()` で異種グラフ用に自動変換**するのが推奨パターンです。手動で各エッジ種に SAGEConv を書くと方向ミスを起こしやすいため避けます。

`scripts/gnn_model.py` の内容：

```python
import torch
import torch.nn as nn
from torch_geometric.nn import SAGEConv, to_hetero


class GNNEncoder(nn.Module):
    """同種グラフ前提のシンプルな2層 GraphSAGE。
    入力次元は (-1, -1) でレイジー初期化（ノード種ごとの次元差を吸収）。"""

    def __init__(self, hidden_dim: int = 64):
        super().__init__()
        self.conv1 = SAGEConv((-1, -1), hidden_dim)
        self.conv2 = SAGEConv((-1, -1), hidden_dim)

    def forward(self, x, edge_index):
        x = self.conv1(x, edge_index).relu()
        x = self.conv2(x, edge_index)
        return x


class EdgeDecoder(nn.Module):
    """ユーザー・映画ペアの内積でスコア（リンクの存在確率の logit）を返す。"""

    def forward(self, z_dict, edge_label_index):
        u = z_dict["user"][edge_label_index[0]]
        m = z_dict["movie"][edge_label_index[1]]
        return (u * m).sum(dim=-1)


class MovieRecommender(nn.Module):
    def __init__(self, data, hidden_dim: int = 64):
        super().__init__()
        # ユーザーは特徴量がないので学習可能な埋め込みを用意
        self.user_emb = nn.Embedding(data["user"].num_nodes, hidden_dim)
        # 映画はジャンル特徴量を hidden_dim に射影
        self.movie_lin = nn.Linear(data["movie"].x.size(-1), hidden_dim)

        encoder = GNNEncoder(hidden_dim)
        # data.metadata() = (node_types, edge_types) を渡すと
        # PyG が自動で各エッジ種ごとの SAGEConv を生成してくれる
        self.encoder = to_hetero(encoder, data.metadata(), aggr="sum")
        self.decoder = EdgeDecoder()

    def forward(self, data):
        x_dict = {
            "user": self.user_emb(torch.arange(data["user"].num_nodes, device=self.user_emb.weight.device)),
            "movie": self.movie_lin(data["movie"].x),
        }
        z_dict = self.encoder(x_dict, data.edge_index_dict)
        edge_label_index = data["user", "rates", "movie"].edge_label_index
        return self.decoder(z_dict, edge_label_index)
```

ポイント：

- **`SAGEConv((-1, -1), hidden_dim)`** — 入力次元を `-1` にすると、初回 forward 時に自動で形状を確定します（レイジー初期化）。ノード種ごとに特徴量次元が違っても1つのモデルで扱えます。
- **`to_hetero(encoder, data.metadata())`** — 同種グラフ用に書いた `GNNEncoder` を、`(user, rates, movie)` と `(movie, rev_rates, user)` の両方向に自動展開します。Step 1 で `T.ToUndirected()` を適用したのはこのためです。
- **`nn.Embedding`** — ユーザーは年齢・性別のような特徴量を使わず、ID から学習する埋め込みベクトルを使います（コールドスタートを単純化するため）。

## Step 3: 訓練ループの設計

本書では **フルバッチ訓練** を採用します。

```python
# 正例エッジ（実際の評価）はそのまま使う
pos = train_data["user", "rates", "movie"].edge_label_index   # shape: [2, ~80,000]

# 負例エッジは毎エポック・毎バッチで一様ランダムにサンプリング
neg = torch.stack([
    torch.randint(0, num_users, (num_pos,)),
    torch.randint(0, num_movies, (num_pos,)),
], dim=0)

# 正例ラベル=1, 負例ラベル=0 で BCE 損失を計算
edge_label_index = torch.cat([pos, neg], dim=1)
labels = torch.cat([torch.ones(num_pos), torch.zeros(num_pos)])
```

:::message
**なぜフルバッチか？**

PyG の `LinkNeighborLoader`（隣人サンプリング）は内部的に `pyg-lib` ライブラリを必要とします。`pyg-lib` は Mac Apple Silicon 向けの公式 wheel が PyPI 経由で取得しづらく、初心者がはまる原因になります。

MovieLens 100K は約 100,000 エッジ・2,600 ノードと小さく、フルバッチでも CPU で 1〜2 分で 10 エポック完走します。実用規模で扱う場合のみサンプリングを検討してください。
:::

`scripts/train_gnn.py` の中で、上記のサンプリングをエポックごとに繰り返します。

## 動作確認

モデルが正しく初期化できるか確認：

```bash
uv run python scripts/check_gnn_model.py
```

`check_gnn_model.py` は `data/pyg_data.pt` をロードしてモデルを構築し、グラフ全体に対する forward が通ることを確認します。

```
Model params: 94,656
Output shape: (80000,)
```

パラメータ数と forward の出力形状（訓練エッジ数と一致）が表示されれば OK です。

## 動作確認チェックリスト

- [ ] `prepare_pyg_data.py` が完走し、`data/pyg_data.pt` が作成された
- [ ] `data/pyg_data.pt` の中身が `HeteroData` 形式になっている
- [ ] モデル定義スクリプトがエラーなく実行できた

次章でこのモデルを訓練し、実際の推薦結果を確認します。
