---
title: "Claude Code 実践 — Inventory Management で学ぶ12ステップ"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day1/01_inventory-management/`

サンフランシスコでの2日間、最初のハンズオンで触ったのは Vue3 と FastAPI でできた在庫管理アプリだった。よくある業務システムの題材だ。けれど、画面の出来を眺めるためのセッションではない。題材を素材にして、Claude Code という道具に「自分の手の延長」として馴染んでいくための時間だった。

最初に CLI を叩いた瞬間、それまで自分が抱いていた Claude Code 像が一段ずれた。ターミナルから呼べる賢いコード生成 AI、くらいの印象でいた。実際に席に座ってキーを打ち始めると、そういう浅い理解では太刀打ちできない設計思想がそこにあることが、じわじわとわかってくる。コンテキストをどう設計するか。権限の境界をどう引くか。外側のシステムとどう連携させるか。Claude Code が触らせてくるのは、コード生成というより、自分の働き方の OS そのものだった。

## 触っていた題材

ハンズオンの素材は、ある工場の在庫を管理する小さなフルスタックアプリだ。フロントエンドは Vue 3 (Composition API) + Vite で 3000 番、バックエンドは Python FastAPI で 8001 番、データは `server/data/*.json` を `server/mock_data.py` 経由でメモリにロードしている。ローカルだけで完結する、ちょうどよい大きさのアプリだった。

進め方は 12 ステップに分割されていた。Fork して `claude` を起動し、モデルを選び、`CLAUDE.md` を眺め、Plan Mode で機能を設計し、`/context` と `/compact` でコンテキストの呼吸を整え、Playwright MCP を入れてブラウザ操作を任せ、GitHub App を仕込んで PR を自動レビューさせる。最後の Expert Challenge では、Reports ページに仕込まれた8つ以上のバグをまとめて直す。順を追って小さく成功体験を重ねながら、Claude Code の語彙を一通り身体に入れるための構成だ。

## 何を学んだか

ステップ1から3までは導入で、戸惑いはほとんどなかった。`claude` と打って、`/model` でモデルを選び、「依存関係を入れて開発サーバを立てて、ブラウザで両方開いて」とだけ伝えると、フロントエンドとバックエンドの面倒を見て、ブラウザまで開いてくれる。最初の感動はここではなく、その次にやってきた。

`CLAUDE.md` の存在を意識しはじめたあたりから、見えてくる景色が変わった。プロジェクト直下に置かれたこのファイルは、毎セッションで自動的にコンテキストへ載る「永続指示書」だ。本リポジトリの `day1/01_inventory-management/CLAUDE.md` を眺めると、技術スタック、強制ルール、ハマりどころ、ファイル位置までがコンパクトに書かれている。

```markdown
# CLAUDE.md

Factory Inventory Management System Demo — Full-stack application with
Vue 3 frontend, Python FastAPI backend, and in-memory mock data.

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: Use for Vue 3 frontend features, UI components, ...
  - MANDATORY RULE: ANY time you need to create or significantly modify
    a .vue file, you MUST delegate to vue-expert
- **code-reviewer**: Use after writing significant code to review quality

### MCP Tools
- ALWAYS use GitHub MCP tools (mcp__github__*) for ALL GitHub operations
- ALWAYS use Playwright MCP tools (mcp__playwright__*) for browser testing

## Stack
- Frontend: Vue 3 + Composition API + Vite (port 3000)
- Backend: Python FastAPI (port 8001)

## Common Issues
1. Use unique keys in v-for (not `index`) — use `sku`, `month`, etc.
2. Validate dates before .getMonth() calls
3. Update Pydantic models when changing JSON data structure
```

このファイルが書かれているおかげで、毎回プロンプトで前置きを唱える必要がない。`.vue` ファイルを触るときは vue-expert に委譲する、GitHub 操作は MCP を使う、`v-for` のキーには `index` を使わない。チームの暗黙知が、Claude にとっての「常識」として常駐する。読んでいるうちに、これはコードではなく、自分たちの働き方そのものをコード化したものなのだと気づいた。

`#` で入る Memory Mode は、その「働き方の更新」を会話の途中で即座に固定化するための入り口だった。気になったルールをその場で `#` のあとに書き留めると、トークンを一切消費せず `CLAUDE.md` に追記される。直接エディタで編集する、Claude に編集を頼む、Memory Mode で書く。三つの選択肢があり、使い分ける筋肉が育っていく感覚があった。

| 方法 | トークン消費 | 用途 |
| --- | --- | --- |
| `#` キーで Memory Mode | ゼロ | 「これからは常にこうしてほしい」をその場で記録する |
| エディタで直接編集 | ゼロ | 構造的に整理したい / レビュー対象にしたい |
| プロンプトで Claude に編集させる | あり | 文言整理・章立て再構築など、Claude の判断を借りたいとき |

Plan Mode に切り替えたときも、似たような感触があった。`Shift+Tab` でモードを切り替えると、Claude はいきなりコードを書きはじめない。どのファイルを触り、どこに依存があり、どう変えるか。設計の手前で立ち止まる。在庫管理アプリに「予算スライダー付きの再発注タブ」を追加するくらいの複雑さになると、Plan Mode に入っているかどうかで、出来上がりの品質も、自分の納得感も、明確に変わった。書かせる前に合意する。コードレビューの後段ではなく前段で対話する。この順序の入れ替えが、ハンズオンで一番強く身体に残った感触だ。

コンテキスト管理の話も、抽象的な議論で終わらず、画面の数値として見えたのが効いた。`/context` を叩くと、会話・ファイル・ツール定義それぞれがどれだけ枠を食っているかが表示される。Playwright MCP を入れる前と後で叩いてみると、ツール定義の重みだけで目に見えて余白が削れる。文脈は無料ではない、という当たり前のことを、計器付きで体験させられた。残量が苦しくなったら `/compact` で要約圧縮できるが、`/compact keep the details of the restocking feature` のように残したい情報を指定できるのも気が利いている。

Playwright MCP の追加は、Claude を CLI の外に連れ出す最初のステップだった。

```bash
# Bash モード ( ! ) から実行
! claude mcp add playwright npx @playwright/mcp@latest
```

これだけでブラウザの自動操作が手に入る。「開発サーバを立てて、localhost:3000 に行って、ダッシュボードのスクリーンショットを撮って、主要タブをクリックして回って」と頼むと、Claude が黙々と画面遷移を試しながら結果を返してくる。ここで「外部システムに動作の場を広げる」感覚を初めて持った。Claude が触れる世界は、ファイルシステムの内側だけではない。

GitHub App の `/install-github-app` まで進むと、感覚はさらに変わる。ローカルで動く Claude Code と、クラウドで動く Claude Code Review が、PR を媒介に対話しはじめる。`@claude このバグの原因を調べて` と PR コメントに書けば、クラウド側のエージェントが応答する。レビュー待ちのアイドル時間が、別のエージェントの労働時間に置き換わる。一人の開発者として動いていたつもりが、いつの間にか小さなチームを指揮する側に回っていた。

最後のステップ12で並ぶ Skills、Subagents、Hooks、Plugins、Agent Teams、Worktrees は、それぞれが「単発の便利機能」ではなく、働き方の別の軸を担当している。Skills は再利用可能な手順書、Subagents は専門特化したペアプロ相手、Hooks は規律を強制する番人、Worktrees は実験を本流から隔離する避難所だ。一つひとつをハンズオン中にすべて使い倒すのは無理でも、輪郭をつかんでおけば、現場に戻ったあと「ここはあの機能を使う場面だ」と当てがついていく。

## 前提が崩れた瞬間

ハンズオンの途中で、自分のなかの思い込みがいくつか崩れた。

ひとつめは「MCP は多いほど良い」という素朴な発想だ。実際に Playwright MCP を入れる前後で `/context` を眺めると、ツール定義だけでまとまった量のコンテキストが消える。経験者の口からは「15 個前後で context lock が起きる」という言葉が出てきた。プロジェクトごとに `.mcp.json` を絞り、使わない MCP は引き算する。足し算より引き算が効く道具だった。

ふたつめは「Skill の description は雰囲気で書けばいい」という思い込みだ。Skills は progressive disclosure、つまり必要になった瞬間に本文が開かれる設計で動く。入口の description が曖昧だと、Claude は呼び出すべき場面を判別できず、結果としてそのスキルは死蔵される。

```text
Bad : description: General-purpose helper for Vue stuff
Good: description: Use when writing or modifying .vue files in client/src/views to refactor inline styles into Tailwind utility classes.
```

何のために、いつ、どんな入力に対して使うのか。description は索引でありメタデータだ、と頭の中で位置づけが変わった。

みっつめは、Skill や CLAUDE.md に「最高の頭脳として振る舞え」「経験豊富なシニアエンジニアになりきれ」と書けば賢くなる、という錯覚だ。気合系の主観的指示はほとんど効かない。Claude が反応するのは「Vue 3 Composition API の `<script setup>` 構文で書く」「`ref` ではなく `computed` を返す」のような、観測可能で再現可能な制約だった。曖昧な賞賛より、具体的な制約。気持ちで盛らずに、ルールで縛る。これは Claude に限らず、人間相手のドキュメントでも同じだったな、と苦笑いした。

四つめは、Reports ページのバグを直す Expert Challenge で起きた。

```text
The Reports page (/reports) has multiple bugs compared to the rest of the app.
I found that:
1. It doesn't translate to Japanese when I switch languages
2. It ignores the global filter bar
3. It spams the browser console with debug logs

Investigate the Reports page code, identify ALL the issues (there are more
than these 3), and fix them. Look at how other pages like Dashboard and
Orders are implemented for reference.
```

ここで Claude に渡しているのは、バグの列挙ではなく「比較対象」だ。`Dashboard.vue` と `Orders.vue` という動いている参照実装を明示することで、タスクは「未知のバグを当てる」から「動く実装と壊れた実装の差分を埋める」に変わる。Claude は自分が挙げた3つの問題を超えて、8つ以上のバグを淡々と拾って直してくれた。AI に問題を渡すのではなく、AI が比較できる正解を渡す。プロンプトのコツというより、ペアプロにおける情報設計の問題だと感じた。

## 押さえておきたいかたち

ハンズオンで何度も立ち返った「型」をいくつか書き残しておく。

`CLAUDE.md` の構造は、覚えるほどのものではないが、毎回ここに収まると気が楽だ。

```markdown
# CLAUDE.md

<1〜2行のプロジェクト概要>

## Critical Tool Usage Rules
### Subagents
- **<agent-name>**: いつ使うか / 強制ルール
### MCP Tools
- ALWAYS use <mcp> for <用途>

## Stack
- Frontend / Backend / Data の最小情報

## Quick Start
<コピペで動く起動コマンド>

## Key Patterns
<このプロジェクト固有のデータフロー・反応性パターン>

## Common Issues
1. <ハマりどころ1>
2. <ハマりどころ2>

## File Locations
- <重要ファイルの場所>

## Coding Standards
- <肯定形で1〜3行>
```

Skill の frontmatter は、progressive disclosure を体現するための入口設計だ。description に「いつ起動すべきか」を書き、本文に「呼ばれてから読まれる詳細」を書く。

```markdown
---
name: vue-composition-refactor
description: |
  Use when refactoring Vue 3 Single File Components in client/src/views
  to migrate Options API to Composition API <script setup>. Triggers on
  prompts mentioning "convert to script setup", "refactor to composition
  API", or "migrate Options API". Returns a diff with reactivity preserved.
allowed-tools:
  - Read
  - Edit
  - Bash(npm run *)
---

# Vue Composition Refactor

## When to use
- ファイル拡張子が `.vue` で `<script>` (setup なし) を使っている
- ユーザが Composition API への移行を求めている

## Steps
1. 対象 SFC を Read で取得
2. `data()` / `methods` / `computed` を `ref` / `function` / `computed` に変換
3. テンプレートのバインディングは変更しない
4. `npm run typecheck` で検証
```

Hooks は、Claude が壊れたコードを残さないための「自動の番人」だ。最小例として、`Edit` または `Write` のあとに Prettier をかけるだけでも、品質の底が一段上がる。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATH\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

`prettier` を `eslint`、`mypy`、`tsc` に置き換えれば、自分のスタックに合わせた規律を Claude に課せる。設定一発で、コードレビューの最初の30分が自動化される。

## 現場に持ち帰りたいこと

ハンズオンを離れて、自分のプロジェクトに戻ったときに最初に試したいことが、いくつか頭に残っている。

Worktree を使った並行作業はそのひとつだ。`git worktree` で別ブランチを別ディレクトリに切り出し、本流の作業を続けたまま実験的な改修を回す。Claude に「Worktree を切ってダークモードを試作して、結果を見せて」と頼める軽さは、思っていたより心理的ハードルを下げる。要らなければ Worktree ごと捨てればいい。失敗のコストが下がると、試す回数は素直に増える。

`/context` を「会話の体温計」にする習慣も持ち帰りたい。30 ターンを超えたあたりで一度叩く。80% を超えたら `/compact`、90% を超えたら `/clear` と決めておく。重要な事実は事前に `CLAUDE.md` に逃がす。この三段構えだけで、長丁場のセッションの終盤に品質が崩れる現象を、ずいぶん抑えられそうだ。

GitHub App と `@claude` を PR レビューの一次対応に据えるのも、効果が想像しやすかった。トリビアルな指摘を機械が拾い、人間レビュアは論点が整った状態から議論を始める。レビュアの時間の使い方が変わると、チームの呼吸そのものが変わる。

## もっと深掘りする入口

ハンズオンが終わったあと、自分用のブックマークとして残しておきたかったリンクを並べておく。

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code Best Practices (Anthropic)](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Workshop ライブ版](https://claude-code-workshop.netlify.app/) — 本章のハンズオンの大元
- [Model Context Protocol 公式](https://modelcontextprotocol.io/) — MCP の仕様と SDK
- [Playwright MCP](https://github.com/microsoft/playwright-mcp) — 本章で導入したブラウザ自動化 MCP
- [Claude Code GitHub App](https://docs.anthropic.com/en/docs/claude-code/github) — `/install-github-app` の詳細
- [Skills / Subagents / Hooks の設計ガイド](https://docs.anthropic.com/en/docs/claude-code/skills) — progressive disclosure と権限スコープ

## 章末に

第1章の最後に、IT 部門の役割そのものが変わる、というテイクアウェイを書いた。Claude Code を実際にハンズオンで触ってみて、その変化はもはや概念の話ではなく、キーボードの下に降りてきた具体的な感触になった。コードを書く速度が上がる、という生産性の話ではない。コードを書く前に、何を約束し、何を強制し、どのエージェントに何を委ね、どこで人間が判断するか。その設計こそが、これからの IT の仕事の中心になっていく。Claude Code は、その設計を書き込むためのキャンバスだった。

次章では、ここで芽生えた「自分の働き方を設計する」という感覚を、もう一段スケールさせる。チームで Claude Code をどう運用するのか。共有・監査・コストの管理をどう設計するのか。Developer Platform としての Claude Code の姿に踏み込んでいく。

→ 次章: [03-developer-platform](./03-developer-platform.md)
