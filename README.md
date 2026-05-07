# zenn_images

[Zenn](https://zenn.dev/) の記事・本を GitHub 連携でデプロイするためのリポジトリ。
本リポジトリは Zenn CLI でローカル管理し、`main` への push で自動公開される。

公式ガイド: [Zenn CLI で記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)

---

## ディレクトリ構成

```text
zenn_images/
├── articles/       # 記事 (Markdown) を配置
│   └── <slug>.md
├── books/          # 本を配置 (slug ディレクトリ単位で管理)
│   └── <book-slug>/
│       ├── config.yaml
│       ├── cover.png
│       └── <chapter-slug>.md
├── images/         # 記事内で参照する画像 (記事ごとにサブディレクトリを切る)
│   └── <article-slug>/
│       └── *.png
├── package.json    # zenn-cli を依存に持つ
└── README.md
```

`images/<article-slug>/` のように **記事ごとにサブディレクトリ** を作って画像を置く運用とする (例: [images/aizo/](images/aizo/))。

---

## セットアップ

初回のみ依存関係を入れる。

```bash
npm install
```

`zenn-cli` が `package.json` に固定されているため、グローバルインストールは不要。

---

## 記事の作成フロー

### 1. 新規記事を作成する

```bash
npx zenn new:article
```

ランダムな slug を持つ Markdown ファイルが [articles/](articles/) に生成される。
slug を指定したい場合はオプションで渡す。

```bash
npx zenn new:article --slug my-article-slug --title "タイトル" --type tech --emoji ✨
```

**slug の制約**: `a-z0-9`, `-`, `_` のみ・**12〜50 文字**。

### 2. Front Matter を編集する

各記事の先頭にある YAML フロントマターを記述する。

```yaml
---
title: "記事タイトル" # 必須
emoji: "🍽️" # アイキャッチ絵文字 (1 文字)
type: "tech" # "tech" (技術) or "idea" (アイデア)
topics: ["claudecode", "mcp"] # タグ (最大 5 件)
published: true # false で下書き
published_at: 2026-05-10 09:00 # 予約投稿 (任意, JST)
publication_name: "" # Publication 配下に投稿する場合のみ指定
---
```

| キー               | 説明                                                                        |
| ------------------ | --------------------------------------------------------------------------- |
| `title`            | 記事タイトル                                                                |
| `emoji`            | 一覧で表示されるアイキャッチ用の絵文字 1 文字                               |
| `type`             | `tech`(技術記事) または `idea`(アイデア)。`tech` は `topics` に技術タグ推奨 |
| `topics`           | タグ配列。最大 5 件                                                         |
| `published`        | `true` で公開、`false` で下書き                                             |
| `published_at`     | `YYYY-MM-DD` または `YYYY-MM-DD hh:mm` 形式の予約投稿日時 (JST)             |
| `publication_name` | Publication 配下に投稿するときのみ                                          |

### 3. プレビューする

ローカルで表示確認する。デフォルトは `http://localhost:8000`。

```bash
npx zenn preview
# ポート指定
npx zenn preview --port 3333
```

### 4. 画像を貼る

優先順位はユースケースに応じて以下から選ぶ:

1. **本リポジトリ内に置く (推奨)**
   `images/<article-slug>/foo.png` に配置し、Markdown から相対参照する。

   ```markdown
   ![説明](/images/<article-slug>/foo.png)
   ```

2. Zenn の[画像アップロードページ](https://zenn.dev/dashboard/uploads)にアップロードし発行された URL を貼る
3. Gyazo など外部サービスの URL を貼る

### 5. 公開する

`published: true` を確認したうえで commit & push。
GitHub と連携している Zenn アカウントが自動でデプロイする。

```bash
git add articles/<slug>.md images/<slug>/
git commit -m "docs: add <slug>"
git push
```

---

## 本 (Book) の作成フロー

### 1. 雛形を作成する

```bash
npx zenn new:book
# slug 指定
npx zenn new:book --slug my-book-slug
```

`books/<book-slug>/` に `config.yaml` と雛形チャプターが生成される。

### 2. `config.yaml` を編集する

```yaml
title: "本のタイトル"
summary: "紹介文"
topics: ["aws", "terraform"] # 最大 5 件
published: true
price: 0 # 有料は 200〜5000 (100 円単位)
chapters: # 表示順
  - introduction
  - chapter-1
  - chapter-2
toc_depth: 2 # 目次の深さ 0〜3
```

- 本の slug も **12〜50 文字、`a-z0-9-_`**。
- カバー画像は `cover.png` (推奨 500 × 700 px) を本のディレクトリ直下に置く。
- 1 冊あたり最大 **100 チャプター**。

### 3. チャプターの Front Matter

```yaml
---
title: "チャプタータイトル"
free: true # 有料本で無料公開するチャプターのみ true
---
```

### 4. プレビュー / 公開

記事と同じ。`npx zenn preview` で確認し push でデプロイ。

---

## 更新・削除

- **更新**: ファイルを編集して push するだけで Zenn 側に反映される。
- **削除**: ローカルでファイルを消しても Zenn 側からは消えない。Zenn の[ダッシュボード](https://zenn.dev/dashboard)から削除する。

---

## 参考リンク

- [Zenn CLI で記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Markdown 記法一覧](https://zenn.dev/zenn/articles/markdown-guide)
- [GitHub リポジトリで Zenn のコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)
- [Publication について](https://zenn.dev/zenn/articles/about-zenn-publication)
