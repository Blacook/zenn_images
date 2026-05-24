---
title: "Claude Code 実践 — Inventory Management で学ぶ12ステップ"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day1/01_inventory-management/`

## はじめに — Claude Code を「働き方の設計面」として捉え直す

本章は、Vue 3 + FastAPI 製の在庫管理アプリを題材にした Day1 のハンズオン「Inventory Management」の振り返りである。素材はよくある業務アプリだが、目的は機能を作ることではなく、Claude Code という道具の語彙 (CLAUDE.md / Plan Mode / `/context` / MCP / GitHub App / Skills / Subagents / Hooks / Worktrees) を一巡し、現場で運用に乗せるための「型」を取り出すことにある。

本章で得るものは三つ。一つめは Claude Code のコア概念を結びつける見取り図、二つめは再利用できる設定スニペット (CLAUDE.md / Skill frontmatter / Hooks)、三つめは現場で陥りやすいアンチパターンを言語化したチェック観点である。

## 題材 — 12 ステップで触る Vue3 + FastAPI 在庫管理

ハンズオンの素材は、ある工場の在庫を管理する小さなフルスタックアプリで、フロントエンドは Vue 3 (Composition API) + Vite、バックエンドは Python FastAPI、データは `server/data/*.json` を `server/mock_data.py` 経由でメモリにロードする構成だ。ローカルだけで完結する。

12 ステップは概ね以下の流れで構成される。

- ステップ 1〜3: `claude` 起動、`/model` でモデル選択、依存関係導入とサーバ起動の委任
- ステップ 4〜5: `CLAUDE.md` の整備、`#` キーによる Memory Mode の活用
- ステップ 6〜7: Plan Mode (Shift+Tab) での機能設計、`/context` と `/compact` によるコンテキスト管理
- ステップ 8: Playwright MCP の追加 (`claude mcp add playwright npx @playwright/mcp@latest`)
- ステップ 9: `/install-github-app` で GitHub App と PR レビューを接続
- ステップ 10〜11: Reports ページのバグ修正 (Expert Challenge)
- ステップ 12: Skills / Subagents / Hooks / Plugins / Agent Teams / Worktrees の輪郭整理

触れる API は `/model`, `/context`, `/compact`, `/clear`, `/install-github-app`, `claude mcp add` 等。登場するデータは `inventory.json`, `orders.json`, `suppliers.json`, `transactions.json` で、画面は Dashboard / Orders / Reports / Suppliers などの基本タブで構成される。

## ベストプラクティス・アンチパターン・重要ポイント

### CLAUDE.md は「働き方の永続指示書」として設計する

:::message
**原則**: `CLAUDE.md` は毎セッションで自動的にコンテキストへ載る永続指示書である。技術スタック・強制ルール・ハマりどころ・ファイル位置をコンパクトに記述し、プロンプトでの前置きを不要にする。
:::

:::message alert
**アンチパターン**: ロール宣言や賞賛 (「あなたはシニアエンジニアです」「最高の頭脳で」) を書き連ねる。曖昧で観測不能な指示はモデルの挙動を変えない。
:::

**ハンズオンでの具体例**: `day1/01_inventory-management/CLAUDE.md` には、`vue-expert` への委譲ルール、GitHub/Playwright MCP の使用強制、`v-for` のキーに `index` を使わない等の観測可能な制約が並ぶ。

```markdown
## Critical Tool Usage Rules
### Subagents
- **vue-expert**: Use for Vue 3 frontend features, UI components, ...
  - MANDATORY RULE: ANY time you need to create or significantly modify
    a .vue file, you MUST delegate to vue-expert

### MCP Tools
- ALWAYS use GitHub MCP tools (mcp__github__*) for ALL GitHub operations
- ALWAYS use Playwright MCP tools (mcp__playwright__*) for browser testing

## Common Issues
1. Use unique keys in v-for (not `index`) — use `sku`, `month`, etc.
2. Validate dates before .getMonth() calls
3. Update Pydantic models when changing JSON data structure
```

加えて `#` キーで起動する Memory Mode は、会話中に確定したルールをその場で `CLAUDE.md` に追記する経路で、トークンを消費しない。三つの編集経路を使い分ける。

| 方法                             | トークン消費 | 用途                                           |
| -------------------------------- | ------------ | ---------------------------------------------- |
| `#` キーで Memory Mode           | ゼロ         | 「これからは常にこうしてほしい」を即座に固定化 |
| エディタで直接編集               | ゼロ         | 構造的に整理する / レビュー対象にする          |
| プロンプトで Claude に編集させる | あり         | 文言整理・章立て再構築など判断を借りる         |

### Plan Mode → 設計合意 → 実装の順序

:::message
**原則**: 複雑度のある変更は `Shift+Tab` で Plan Mode に入り、ファイル・依存・変更方針について合意してから実装に進む。コードレビューを前段の対話に置き換える。
:::

:::message alert
**アンチパターン**: 複数ファイルにまたがる機能追加でも、いきなりコード生成させて差分レビューで巻き戻すフロー。手戻りコストが上がり、コンテキストも食う。
:::

**ハンズオンでの具体例**: 在庫管理アプリへ「予算スライダー付きの再発注タブ」を追加する課題では、Plan Mode を経由したかどうかで成果物の構造的な整合性が分かれた。Plan Mode は対象ファイル一覧と影響範囲をテキストで提示してから着手するため、不要な変更や見落としを着手前に潰せる。

### コンテキスト管理は計器を見て扱う

:::message
**原則**: `/context` でコンテキスト使用量を可視化し、80% で `/compact`、90% で `/clear` を発動する閾値運用にする。重要な事実は事前に `CLAUDE.md` 側へ逃がす。
:::

:::message alert
**アンチパターン**: 残量を見ずに長時間セッションを継続し、終盤で品質劣化や応答の脱線が起きてから対処する。
:::

**ハンズオンでの具体例**: Playwright MCP を追加する前後で `/context` を叩くと、ツール定義だけでまとまった量の枠が削られることが数値で見える。経験者の指摘では「15 個前後の MCP で context lock が起きる」とされ、`/compact keep the details of the restocking feature` のように残したい情報を指定して圧縮できる。

### MCP は引き算で設計する

:::message
**原則**: プロジェクトごとに `.mcp.json` を絞り、使う MCP のみを宣言する。グローバル導入は避け、用途が終わったら外す。
:::

:::message alert
**アンチパターン**: 「便利そうだから」で MCP を足し続け、ツール定義の総量でコンテキスト枠を圧迫する。
:::

**ハンズオンでの具体例**: Playwright MCP の追加は、Claude を CLI の外に連れ出す最初のステップとして実演される。

```bash
# Bash モード ( ! ) から実行
! claude mcp add playwright npx @playwright/mcp@latest
```

導入後は「開発サーバを立てて localhost:3000 のダッシュボードのスクリーンショットを撮り、主要タブを巡回する」と頼むだけで自動操作が走る。一方で導入直後の `/context` 差分は明確に大きく、足す前提と外す前提を最初から持っておく必要がある。

### Skills の description は索引である

:::message
**原則**: Skills は progressive disclosure で、必要になった瞬間に本文が開かれる。入口の `description` は「いつ・どんな入力に対して呼び出すか」のメタデータとして書く。
:::

:::message alert
**アンチパターン**: `General-purpose helper for Vue stuff` のような曖昧表現。Claude が呼び出し場面を判定できず、スキルが死蔵される。
:::

**ハンズオンでの具体例**: 良い description と悪い description の対比は次のとおり。

```text
Bad : description: General-purpose helper for Vue stuff
Good: description: Use when writing or modifying .vue files in client/src/views to refactor inline styles into Tailwind utility classes.
```

トリガ条件 (ファイルパス・ユーザ発話パターン)、入力、出力までを description に書き、本文は「呼ばれてから読まれる手順」に専念させる。

### 規律は Hooks で機械化する

:::message
**原則**: format/lint/typecheck などの規律はプロンプトで頼まず、`PostToolUse` フックで自動実行する。Claude が壊れたコードを残せない構造にする。
:::

:::message alert
**アンチパターン**: 「コミット前に prettier をかけて」を毎回プロンプトで頼む。忘れる、忘れさせる、レビューでの指摘が増える。
:::

**ハンズオンでの具体例**: `Edit|Write` の後段で Prettier を流すだけでも、レビュー一次対応の負荷が下がる。

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

`prettier` を `eslint`, `mypy`, `tsc` に置き換えれば、スタックに合わせた規律を Claude に課せる。

### GitHub App とクラウド側エージェント連携

:::message
**原則**: `/install-github-app` で GitHub App を導入し、PR コメント `@claude ...` でクラウド側 Claude Code Review に一次対応を委ねる。ローカル CLI とクラウド側エージェントは PR を媒介として非同期に協働する。
:::

:::message alert
**アンチパターン**: ローカルの Claude Code のみで PR レビューも自分で読む運用。レビュアの待ち時間がボトルネックになる。
:::

**ハンズオンでの具体例**: PR コメントに `@claude このバグの原因を調べて` と書くと、クラウド側エージェントが応答し、ローカルとは別の労働時間が動き出す。ハンズオンではこの非同期連携を、レビュー導線の置き換えとして体験する。

### 比較対象のあるバグ修正は強い（Reports ページ Expert Challenge）

:::message
**原則**: 「未知のバグを当てる」より「動く実装と壊れた実装の差分を埋める」タスク設計のほうが、Claude の網羅性は跳ね上がる。動く参照実装を明示する。
:::

:::message alert
**アンチパターン**: バグの症状だけを列挙して投げる。Claude は申告された項目に集中し、未申告のバグを取り逃がす。
:::

**ハンズオンでの具体例**: Reports ページの Expert Challenge では次のような指示で、申告 3 件以上のバグまで自動で洗い出される。

```text
The Reports page (/reports) has multiple bugs compared to the rest of the app.
I found that:
1. It doesn't translate to Japanese when I switch languages
2. It ignores the global filter bar
3. It spams the browser console with debug logs

Investigate the Reports page code, identify ALL the issues (there are more
than these 3), and fix them. Look at how other pages like ** Dashboard and
Orders are implemented for reference **.
```

ポイントは `Dashboard.vue` / `Orders.vue` という参照実装を明示している点だ。情報設計でタスクの性質を「差分埋め」に変換している。

### Subagents / Plugins / Worktrees の使い分け

:::message
**原則**: 12 ステップ最終盤の機能群はそれぞれ働き方の異なる軸を担う。Skills = 再利用可能な手順書、Subagents = 専門特化したペアプロ相手、Hooks = 規律の番人、Plugins = 機能パッケージ、Agent Teams = 並走するエージェント群、Worktrees = 実験を本流から隔離する避難所。
:::

:::message alert
**アンチパターン**: すべてを「便利機能」として一律に評価し、どれを採用するか場当たり的に選ぶ。
:::

**ハンズオンでの具体例**: 在庫管理アプリでは `vue-expert` Subagent が `.vue` 編集に強制委譲され、Worktrees は触れないが「ダークモード試作を Worktree で隔離する」シナリオが提示される。Hooks はステップ 12 で format/typecheck フックの形で並ぶ。輪郭をつかんでおくと、現場で「ここはこれを使う」と当てがつく。

## 押さえておきたいコード／設定

ハンズオンで何度も立ち返った「型」を抜粋する。

`CLAUDE.md` の標準骨格。

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

Skill の frontmatter は progressive disclosure の入口設計。`description` に発動条件、本文に手順を分離する。

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

PostToolUse Hook の最小例 (前掲)。`prettier` を `eslint`, `mypy`, `tsc` に差し替えて再利用する。

## よくある勘違いと気づき

- **「Claude Code はターミナル上の賢いコード生成 CLI」だと思っていたが、実際には CLAUDE.md / Plan Mode / Hooks / MCP / GitHub App が連動した「働き方の設計面」だった。**コードを書く速度の話ではなく、何を約束し、何を強制し、誰に委ねるかを設計する道具だ。
- **「MCP は多いほど良い」と思っていたが、足し算より引き算が効く道具だった。**Playwright MCP を入れる前後で `/context` を眺めると、ツール定義だけでコンテキスト枠が目に見えて削れる。15 個前後で context lock が起きるという経験則も腑に落ちた。
- **「Skill の description は雰囲気で書けばいい」と思っていたが、description は索引でありメタデータだった。**入口が曖昧だとスキルは死蔵される。「いつ・どんな入力で・何を返すか」を書くものだった。
- **「最高の頭脳として振る舞え」と書けば賢くなると錯覚していたが、Claude は気合系の主観的指示にはほぼ反応しなかった。**反応するのは「`<script setup>` で書く」「`ref` ではなく `computed` を返す」のような観測可能な制約だった。気持ちで盛らずに、ルールで縛る。
- **「バグ修正は症状を渡すのがプロンプトのコツ」だと思っていたが、効くのは『比較対象を渡す』ことだった。**Reports ページの課題で動く参照実装 (`Dashboard.vue`, `Orders.vue`) を明示しただけで、Claude は申告 3 件を超える 8 件以上のバグを淡々と直した。タスクを「未知のバグ当て」から「差分埋め」に作り変える情報設計だった。
- **「Plan Mode はコード生成を遅らせるだけ」だと思っていたが、書かせる前に合意する順序の入れ替えが、出来上がりの品質と納得感を明確に変えた。**設計の手前で立ち止まる時間は、後工程の手戻りより安い。

## 現場に持ち帰りたいこと

- **Worktree を「失敗のコストを下げる装置」として使う。**`git worktree` で別ブランチを別ディレクトリに切り出し、本流の作業を続けたまま実験的な改修を Claude に任せる。要らなくなれば Worktree ごと捨てる。試行回数が増える設計を、機能ではなく運用として定着させる。
- **`/context` を会話の体温計にする。**30 ターン超で叩き、80% で `/compact`、90% で `/clear` の三段構えを習慣化する。重要事実は事前に `CLAUDE.md` へ逃がす。
- **Hooks で規律を機械化する。**プロジェクト初期に format/typecheck/lint の `PostToolUse` を整え、プロンプトで頼まなくても規律が回る土台を最初に置く。
- **CLAUDE.md と Skill description を「観測可能な制約」で書き直す。**気合系の表現を一掃し、ファイルパス・関数名・構文要件で書く。

## もっと深掘りする入口

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code Best Practices (Anthropic)](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Workshop ライブ版](https://claude-code-workshop.netlify.app/) — 本章のハンズオンの大元
- [Model Context Protocol 公式](https://modelcontextprotocol.io/) — MCP の仕様と SDK
- [Playwright MCP](https://github.com/microsoft/playwright-mcp) — 本章で導入したブラウザ自動化 MCP
- [Claude Code GitHub App](https://docs.anthropic.com/en/docs/claude-code/github) — `/install-github-app` の詳細
- [Skills / Subagents / Hooks の設計ガイド](https://docs.anthropic.com/en/docs/claude-code/skills) — progressive disclosure と権限スコープ

## 章末 — Claude Code は働き方を書き込むキャンバスだ

第1章で触れた「IT 部門の役割そのものが変わる」というtakeawayは、ハンズオンを経てキーボードの下に降りてきた。コードを書く速度の話ではなく、何を約束し、何を強制し、どのエージェントに何を委ね、どこで人間が判断するか。その設計こそが、これからの IT の仕事の中心になる。Claude Code は、その設計を書き込むためのキャンバスである。

次章では、ここで芽生えた「自分の働き方を設計する」感覚を一段スケールさせる。チームでの運用、共有・監査・コスト管理を含む Developer Platform としての Claude Code に踏み込んでいく。

→ 次章: [03-developer-platform](./03-developer-platform.md)
