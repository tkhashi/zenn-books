# Zenn コンテンツ管理

このリポジトリは [Zenn](https://zenn.dev) の記事・本を GitHub で管理するためのものです。
`zenn-cli` を使ってローカルで執筆し、GitHub push で自動デプロイされます。

## よく使うコマンド

```bash
npx zenn preview                        # プレビュー (localhost:8000)
npx zenn new:article                    # 記事を新規作成
npx zenn new:article --slug <slug>      # slug を指定して記事を新規作成
npx zenn new:book                       # 本を新規作成
npx zenn new:book --slug <slug>         # slug を指定して本を新規作成
```

---

## 記事 (articles/)

### ファイル配置

```
articles/
└── <slug>.md
```

### slug ルール

- 12〜50文字
- 使用可能文字: `a-z`, `0-9`, `-`, `_`

### frontmatter

```yaml
---
title: ""          # 記事タイトル（必須）
emoji: "😸"        # サムネイル用の絵文字1文字（必須）
type: "tech"       # "tech"（技術記事）または "idea"（アイデア）（必須）
topics: []         # タグ（任意、最大5個） 例: ["go", "typescript"]
published: false   # true = 公開、false = 下書き（必須）
published_at: ""   # 公開日時（任意）YYYY-MM-DD または YYYY-MM-DD HH:mm (JST)
---
```

### 注意事項

- **削除はダッシュボードから行う** — ファイルを削除しても Zenn 上の記事は消えない
- `published_at` は一度設定すると変更不可

---

## 本 (books/)

### ディレクトリ構成

```
books/
└── <slug>/
    ├── config.yaml       # 本の設定（必須）
    ├── cover.png         # カバー画像（必須）: 500×700px, JPEG/PNG
    ├── chapter1.md
    └── chapter2.md
```

### slug ルール

- 12〜50文字
- 使用可能文字: `a-z`, `0-9`, `-`, `_`

### config.yaml

```yaml
title: "本のタイトル"
summary: "本の説明（有料本でも表示される）"
topics: ["tag1", "tag2"]   # 最大5個
published: false           # true = 公開
price: 0                   # 0（無料）または 200〜5000（¥、100円単位）
toc_depth: 2               # 目次の深さ 0〜3
chapters:
  - chapter1
  - chapter2
```

### チャプターファイル

```yaml
---
title: "チャプタータイトル"   # 必須
free: true                   # 任意（有料本の無料公開チャプター）
---
```

- チャプタータイトル: 最大70文字
- ファイル名: 1〜50文字、`a-z`, `0-9`, `-`, `_`
- チャプター上限: 100個
- 順序指定の方法は2通り:
  1. `config.yaml` の `chapters:` リストに記述
  2. ファイル名に数字プレフィックス (`1.intro.md`, `2.setup.md`) を付け `chapters: []` にする

### 注意事項

- **削除はダッシュボードから行う** — ディレクトリを削除しても Zenn 上の本は消えない

---

## デプロイ

- GitHub の連携ブランチへ push すると自動デプロイ
- デプロイをスキップ: コミットメッセージに `[ci skip]` または `[skip ci]` を含める
- デプロイ状況の確認: https://zenn.dev/dashboard/deploys

---

## 参考

- [Zenn CLI 使い方ガイド](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Zenn ダッシュボード](https://zenn.dev/dashboard)
