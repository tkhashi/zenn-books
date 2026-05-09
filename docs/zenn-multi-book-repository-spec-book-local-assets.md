# 複数 Zenn Book 管理リポジトリ基盤 仕様書

## 1. 目的

本仕様書は、複数の Zenn Book を 1 つの執筆用 GitHub リポジトリで管理し、Zenn 連携リポジトリへ自動同期するためのリポジトリ基盤を定義する。

主目的は以下の通り。

- 複数 book を単一リポジトリで安全に管理する
- book 単位で差分同期し、他 book への意図しない上書き・削除を防止する
- book 本体と book 固有アセットを同一ディレクトリ配下で管理する
- Pull Request ベースでレビュー・プレビュー・検証を行う
- main ブランチへの merge 後に Zenn 連携リポジトリへ自動デプロイする
- 将来的な book 追加・削除・移管に耐えられる構成にする

## 2. 前提

### 2.1 Zenn 側の前提

Zenn の GitHub 連携では、登録ブランチへの push または Pull Request の merge により Zenn 側の同期が開始される。

Zenn Book は通常、以下の形式で管理する。

```txt
books/
  <book-slug>/
    config.yaml
    chapter-1.md
    chapter-2.md
```

`config.yaml` の `chapters` に、公開対象となる chapter の slug を順序付きで定義する。

### 2.2 リポジトリ前提

本仕様では、以下の 2 リポジトリ構成を標準とする。

| 種別 | 用途 | 例 |
|---|---|---|
| 執筆用リポジトリ | 原稿、レビュー、CI、管理用設定を保持する | `zenn-books-source` |
| Zenn 連携リポジトリ | Zenn と GitHub 連携する公開同期先 | `zenn.dev` |

執筆用リポジトリを直接 Zenn に連携する方式も可能だが、本仕様では「執筆用」と「Zenn 連携先」を分離する。

理由は以下。

- 管理用ファイル、CI 設定、テンプレート、内部メモを Zenn 連携対象から分離できる
- book 単位同期を制御しやすい
- Zenn 連携リポジトリを公開用成果物として単純化できる
- 将来的に book ごとの公開制御や移管がしやすい

## 3. ディレクトリ構成

### 3.1 執筆用リポジトリ

```txt
.
├── books/
│   ├── book-a/
│   │   ├── config.yaml
│   │   ├── introduction.md
│   │   ├── chapter-1.md
│   │   └── images/
│   │       ├── architecture.png
│   │       └── deployment-flow.png
│   ├── book-b/
│   │   ├── config.yaml
│   │   ├── introduction.md
│   │   ├── chapter-1.md
│   │   └── images/
│   │       └── module-boundary.png
│   └── README.md
├── articles/
│   └── .gitkeep
├── .github/
│   ├── workflows/
│   │   ├── validate.yml
│   │   └── deploy-book.yml
│   └── pull_request_template.md
├── scripts/
│   ├── detect-changed-books.sh
│   └── validate-book-config.js
├── package.json
├── README.md
└── SPEC.md
```

### 3.2 Zenn 連携リポジトリ

```txt
.
├── books/
│   ├── book-a/
│   │   ├── config.yaml
│   │   ├── introduction.md
│   │   └── chapter-1.md
│   └── book-b/
│       ├── config.yaml
│       ├── introduction.md
│       └── chapter-1.md
└── articles/
```

Zenn 連携リポジトリには、原則として Zenn が読む必要のあるファイルのみ配置する。

## 4. Book 管理ルール

### 4.1 book slug

book のディレクトリ名は Zenn 上の book slug と一致させる。

```txt
books/<book-slug>/
```

命名規則:

- 小文字英数字とハイフンのみ使用する
- スペース、アンダースコア、日本語、記号は使用しない
- 一度公開した slug は原則変更しない

例:

```txt
books/aws-cdk-handbook/
books/typescript-api-design/
books/ddd-practical-guide/
```

### 4.2 `config.yaml`

各 book は必ず `config.yaml` を持つ。

例:

```yaml
title: "AWS CDK Handbook"
summary: "AWS CDK を実務で使うための設計・運用ガイド"
topics: ["aws", "cdk", "typescript"]
published: true
price: 0
chapters:
  - introduction
  - environment
  - construct-design
  - deployment
```

必須項目:

| 項目 | 必須 | 説明 |
|---|---:|---|
| `title` | Yes | book タイトル |
| `summary` | Yes | book 概要 |
| `topics` | Yes | Zenn 上のトピック |
| `published` | Yes | 公開状態 |
| `price` | Yes | 価格 |
| `chapters` | Yes | chapter slug の順序 |

### 4.3 chapter ファイル

chapter は `books/<book-slug>/*.md` として配置する。

```txt
books/aws-cdk-handbook/introduction.md
books/aws-cdk-handbook/environment.md
```

chapter slug はファイル名から `.md` を除いた値とする。

`config.yaml` の `chapters` に存在しない chapter は Zenn Book の目次に含めない。

### 4.4 book 固有アセット

画像などの book 固有アセットは、各 book ディレクトリ配下の `images/` に配置する。

```txt
books/
  aws-cdk-handbook/
    config.yaml
    introduction.md
    chapter-1.md
    images/
      architecture.png
      deployment-flow.png
```

Markdown からの参照例:

```md
![アーキテクチャ図](./images/architecture.png)
```

原則として、book 外部の共通 `assets/` ディレクトリは使用しない。

理由は以下。

- book 単位で原稿と画像を同時に確認できる
- Pull Request の差分確認が容易になる
- book 単位同期時に画像も一緒に同期できる
- book 削除・移管時に必要ファイルをまとめて扱える
- 他 book の画像を誤って変更するリスクを下げられる

### 4.5 任意の補助ディレクトリ

必要に応じて、book 配下に補助ディレクトリを作成してよい。

```txt
books/
  aws-cdk-handbook/
    config.yaml
    introduction.md
    chapter-1.md
    images/
    snippets/
    diagrams/
```

ただし、chapter `.md` は原則として book 直下に配置する。

推奨:

```txt
books/<book-slug>/<chapter-slug>.md
books/<book-slug>/images/*.png
```

非推奨:

```txt
books/<book-slug>/chapters/<chapter-slug>.md
```

chapter ファイルを `chapters/` 配下に置く構成は、Zenn CLI / GitHub 連携との互換性確認が必要になるため、本仕様では採用しない。

## 5. ブランチ戦略

### 5.1 ブランチ

| ブランチ | 用途 |
|---|---|
| `main` | デプロイ対象 |
| `feature/<topic>` | 執筆・修正作業 |
| `book/<book-slug>/<topic>` | book 単位の作業 |

### 5.2 基本フロー

1. `book/<book-slug>/<topic>` ブランチを作成
2. 原稿・画像を編集
3. Pull Request を作成
4. CI で構文・設定を検証
5. レビュー
6. `main` に merge
7. GitHub Actions が変更 book を検出
8. 対象 book ディレクトリのみ Zenn 連携リポジトリへ同期
9. Zenn 側の GitHub 連携により同期・デプロイ

## 6. GitHub Actions 設計

## 6.1 設計方針

標準方針は「book 単位同期」とする。

禁止する同期方式:

```yaml
source-directory: books
destination-directory: books
```

上記は `books/` 全体を同期対象にするため、転送先に存在する他 book の削除・上書きリスクがある。

推奨する同期方式:

```yaml
source-directory: books/<book-slug>
destination-directory: books/<book-slug>
```

この方式では、更新対象を book 単位に限定できる。

book 固有画像も `books/<book-slug>/images/` 配下に存在するため、別途 assets 用 job は不要とする。

## 6.2 validate workflow

Pull Request 時に実行する。

`.github/workflows/validate.yml`

```yaml
name: validate

on:
  pull_request:
    paths:
      - "books/**"
      - "articles/**"
      - "package.json"
      - "package-lock.json"
      - ".github/workflows/**"

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - run: npm ci

      - name: Validate Zenn contents
        run: npx zenn-cli preview --no-watch
```

補足:

- `zenn-cli preview --no-watch` は CI での完全検証用途としてはプロジェクトに合わせて調整する
- 必要に応じて markdownlint、textlint、独自 config 検証を追加する

## 6.3 deploy workflow

`main` への push 時に、変更された book のみ同期する。

`.github/workflows/deploy-book.yml`

```yaml
name: deploy book to zenn repository

on:
  push:
    branches:
      - main
    paths:
      - "books/**"

permissions:
  contents: read

jobs:
  detect:
    runs-on: ubuntu-latest
    timeout-minutes: 5

    outputs:
      books: ${{ steps.detect.outputs.books }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - id: detect
        name: Detect changed books
        shell: bash
        run: |
          set -euo pipefail

          changed_files="$(git diff --name-only HEAD^ HEAD)"

          books="$(
            echo "$changed_files" |
              awk -F/ '/^books\/[^/]+\// {print $2}' |
              sort -u |
              jq -R -s -c 'split("\n") | map(select(length > 0))'
          )"

          echo "books=$books" >> "$GITHUB_OUTPUT"

  deploy:
    needs: detect
    if: needs.detect.outputs.books != '[]'
    runs-on: ubuntu-latest
    timeout-minutes: 10

    strategy:
      fail-fast: false
      matrix:
        book: ${{ fromJson(needs.detect.outputs.books) }}

    steps:
      - uses: actions/checkout@v4

      - id: generate_token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.ZENN_SYNC_APP_ID }}
          private-key: ${{ secrets.ZENN_SYNC_APP_PRIVATE_KEY }}
          repositories: zenn.dev
          permission-contents: write

      - name: Push book to Zenn repository
        uses: Songmu/action-push-to-another-repository@v2
        with:
          token: ${{ steps.generate_token.outputs.token }}
          source-directory: books/${{ matrix.book }}
          destination-repository: YOUR_NAME/zenn.dev
          destination-directory: books/${{ matrix.book }}
          destination-branch: main
          commit-message: "sync book: ${{ matrix.book }}"
```

## 6.4 book 配下 images の扱い

`books/<book-slug>/images/` は book 本体と同じ同期単位に含める。

そのため、以下の同期のみで原稿と画像をまとめて Zenn 連携リポジトリへ転送できる。

```yaml
source-directory: books/${{ matrix.book }}
destination-directory: books/${{ matrix.book }}
```

assets 専用 job は作成しない。

## 7. GitHub App / Token 設計

### 7.1 推奨方式

同期先リポジトリへの write 権限は、GitHub App で付与する。

必要な権限:

| 権限 | 値 |
|---|---|
| Repository contents | Read and write |
| Metadata | Read |

GitHub Actions secrets:

| Secret | 内容 |
|---|---|
| `ZENN_SYNC_APP_ID` | GitHub App ID |
| `ZENN_SYNC_APP_PRIVATE_KEY` | GitHub App private key |

### 7.2 Personal Access Token を避ける理由

- 個人アカウントに依存する
- 退職・異動時に運用が壊れやすい
- 権限範囲が過剰になりやすい
- ローテーション責任が曖昧になりやすい

## 8. CI 検証項目

### 8.1 必須検証

| 検証 | 内容 |
|---|---|
| YAML 構文 | `config.yaml` が parse できる |
| 必須項目 | `title`, `summary`, `topics`, `published`, `price`, `chapters` が存在する |
| chapter 存在確認 | `chapters` に記載された `.md` が存在する |
| 未参照 chapter 警告 | `config.yaml` にない `.md` を検出する |
| slug 命名 | book slug / chapter slug が命名規則に合う |
| 画像リンク検証 | `./images/...` の参照先が存在する |
| Markdown lint | 見出し、コードブロック、表記揺れを検出する |

### 8.2 推奨 npm scripts

`package.json`

```json
{
  "scripts": {
    "preview": "zenn-cli preview",
    "lint:md": "markdownlint "books/**/*.md" "articles/**/*.md"",
    "lint:text": "textlint "books/**/*.md" "articles/**/*.md"",
    "validate:books": "node scripts/validate-book-config.js",
    "validate": "npm run validate:books && npm run lint:md && npm run lint:text"
  },
  "devDependencies": {
    "zenn-cli": "^0.2.0",
    "markdownlint-cli": "^0.43.0",
    "textlint": "^14.0.0"
  }
}
```

実際のバージョンは導入時点の最新版に合わせて固定する。

## 9. 同期仕様

### 9.1 同期単位

同期単位は `books/<book-slug>` とする。

変更検出対象:

```txt
books/<book-slug>/**
```

画像も book 配下に含めるため、画像変更も同じ book の変更として検出される。

### 9.2 削除仕様

book 削除は自動化しない。

理由:

- 誤削除時の影響が大きい
- Zenn 側の投稿削除はダッシュボード操作も関係する
- GitHub から消しただけでは Zenn 側の公開状態と不整合が起こり得る

book を削除・非公開化する場合は以下の手順とする。

1. `config.yaml` の `published: false` を設定
2. main に merge
3. Zenn 側の反映を確認
4. Zenn ダッシュボードで必要な削除操作を行う
5. 執筆用リポジトリから book ディレクトリを削除
6. Zenn 連携リポジトリから対象 book を手動または専用 workflow で削除

### 9.3 全体同期の扱い

`books/` 全体同期は原則禁止とする。

例外:

- 初回セットアップ時
- 全 book の一括移行時
- Zenn 連携リポジトリを完全に執筆用リポジトリの mirror として扱う場合

例外実行時は、事前に同期先リポジトリの backup branch を作成する。

```bash
git checkout main
git pull
git checkout -b backup/before-full-sync-YYYYMMDD
git push origin backup/before-full-sync-YYYYMMDD
```

## 10. Pull Request ルール

### 10.1 PR 単位

原則として、1 PR で変更する book は 1 つに限定する。

許容する例外:

- 共通 assets の修正
- 全 book に適用する textlint ルール変更
- Zenn CLI / CI 設定変更
- typo 一括修正

### 10.2 PR テンプレート

`.github/pull_request_template.md`

```md
## 対象 book

- [ ] book-a
- [ ] book-b
- [ ] その他

## 変更内容

## 確認項目

- [ ] `config.yaml` の chapters と chapter ファイルが一致している
- [ ] ローカル preview で表示確認した
- [ ] `images/` 配下の画像リンクが壊れていない
- [ ] 公開状態 `published` が意図通りである
- [ ] 他 book に不要な変更を入れていない
```

## 11. 初期セットアップ手順

### 11.1 執筆用リポジトリ作成

```bash
mkdir zenn-books-source
cd zenn-books-source
git init
npm init -y
npm install --save-dev zenn-cli markdownlint-cli textlint
npx zenn-cli init
```

### 11.2 book 作成

```bash
mkdir -p books/aws-cdk-handbook/images
touch books/aws-cdk-handbook/config.yaml
touch books/aws-cdk-handbook/introduction.md
```

### 11.3 Zenn 連携リポジトリ作成

```bash
mkdir zenn.dev
cd zenn.dev
git init
mkdir books articles
touch articles/.gitkeep
git add .
git commit -m "initial zenn repository"
git remote add origin git@github.com:YOUR_NAME/zenn.dev.git
git push -u origin main
```

### 11.4 Zenn 側設定

Zenn のダッシュボードから GitHub リポジトリを連携し、同期対象ブランチを `main` に設定する。

## 12. 運用設計

### 12.1 book 追加

1. `books/<new-book-slug>/` を作成
2. `config.yaml` を作成
3. `images/` を作成
4. 最低 1 chapter を作成
5. `published: false` で PR 作成
6. CI / preview 確認
7. main merge
8. Zenn 連携リポジトリへ同期
9. 公開準備完了後に `published: true` へ変更

### 12.2 book 公開

```yaml
published: true
```

main に merge すると Zenn 側へ同期される。

### 12.3 book 非公開

```yaml
published: false
```

main に merge して Zenn 側の反映を確認する。

### 12.4 chapter 追加

1. `books/<book-slug>/<chapter-slug>.md` を作成
2. `config.yaml` の `chapters` に追加
3. preview で目次順を確認
4. PR 作成


### 12.5 画像追加

1. `books/<book-slug>/images/<image-name>` を追加
2. chapter から `./images/<image-name>` で参照
3. preview で表示確認
4. PR 作成

例:

```md
![デプロイフロー](./images/deployment-flow.png)
```

### 12.6 chapter 削除

1. `config.yaml` の `chapters` から削除
2. 該当 `.md` を削除または `_drafts` 相当の退避場所へ移動
3. PR で削除意図を明記する

## 13. リスクと対策

| リスク | 内容 | 対策 |
|---|---|---|
| 他 book の上書き | `books/` 全体同期により他 book が変更される | book 単位同期に限定する |
| 他 book の削除 | 同期元に存在しない book が同期先から消える | 全体同期禁止、削除 workflow を分離 |
| chapter 欠落 | `config.yaml` の `chapters` と実ファイルが不一致 | CI で検出 |
| 画像リンク切れ | 画像移動・削除で参照が壊れる | `./images/...` を CI で検証 |
| 公開事故 | `published: true` のまま未完成 book を merge | PR checklist、初期値 false |
| 個人 token 依存 | PAT 所有者変更で同期不能 | GitHub App を使用 |
| Zenn 仕様変更 | CLI / 連携仕様の変更 | package lock、定期的な検証 |

## 14. 推奨設定

### 14.1 branch protection

`main` ブランチに以下を設定する。

- Pull Request 必須
- CI 成功必須
- 直接 push 禁止
- 1 名以上の review 必須
- 管理者にも適用

### 14.2 CODEOWNERS

`.github/CODEOWNERS`

```txt
/books/aws-cdk-handbook/ @owner-a
/books/typescript-api-design/ @owner-b
/.github/workflows/ @repo-admin
/scripts/ @repo-admin
```

### 14.3 Dependabot

`.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
```

## 15. 完了条件

本リポジトリ基盤は、以下を満たした時点で初期構築完了とする。

- 複数 book を `books/<book-slug>` 配下で管理できる
- book 固有画像を `books/<book-slug>/images/` 配下で管理できる
- PR 時に validate workflow が実行される
- main merge 時に変更 book のみ Zenn 連携リポジトリへ同期される
- 他 book に対する意図しない上書き・削除が発生しない
- Zenn 側で対象 book の同期結果を確認できる
- README に執筆・追加・公開手順が記載されている

## 16. 参考情報

- Zenn: GitHub リポジトリ連携では、登録ブランチへの push または Pull Request merge により同期が開始される
- Zenn: 公開には `published: true` を設定し、連携ブランチへ push する
- Zenn: デプロイ履歴とエラーはダッシュボードで確認する
- Zenn CLI: ローカル preview による表示確認が可能
- Songmu/action-push-to-another-repository: 特定ディレクトリを別リポジトリの指定ディレクトリへ push できる

## 17. 今後の拡張候補

- 変更 book 検出 script の TypeScript 化
- `config.yaml` schema の JSON Schema 化
- textlint プリセットの整備
- book ごとの preview URL 通知
- PR コメントで変更 book 一覧を自動表示
- GitHub Releases と連動した book 更新履歴生成
- Zenn 連携リポジトリの同期結果監視
