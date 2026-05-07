---
title: Claude Codeの全機能を「飲食店の厨房」で完全理解する
emoji: "🍽️"
type: "tech"
topics: ["claudecode", "agentteams", "skills", "mcp", "subagents"]
published: true
---

## はじめに

Claude Codeには、サブエージェント、Skills、MCP、Hooks、Plugins……と多くの拡張機能があります。公式ドキュメントを読んでも「結局どれをいつ使うの？」と混乱しがちです。

そこで本記事では、**Claude Codeの機能体系を公式ドキュメントの構造に沿って6つのレイヤーに分類**し、それぞれを「飲食店の厨房」に例えて整理します。

:::message
本記事は2026年4月時点の情報に基づいています。Agent Teams等の実験的機能を含みます。
公式ドキュメント「[Extend Claude Code](https://code.claude.com/docs/en/features-overview)」を併せて読むことを推奨します。
:::

## 全体像：6つのレイヤーで理解するClaude Code

Claude Codeの全機能は、**エージェントループ（=料理長の思考→判断→実行サイクル）を中心に、6つのレイヤー**で整理できます。上位レイヤーが下位レイヤーを利用する関係です。

| レイヤー | Claude Codeの機能群 | 厨房での例え | 役割 |
|---|---|---|---|
| **1. コア** | Main Agent, Tools, Context Window, Compaction | 料理長 + 調理器具 + 作業台 | 全機能の土台 |
| **2. エージェント** | Subagents (Built-in / Custom), Agent Teams | 副料理長・専門料理人・バンケット班 | 委任と並列化 |
| **3. 知識** | CLAUDE.md, Skills, `.claude/rules/`, Memory | ハウスルール・専門レシピ・共通規約・常連ノート | Claudeが知っていること |
| **4. 外部接続** | MCP (Model Context Protocol) | 仕入れ業者 | 厨房の外とつながる |
| **5. 自動化と安全** | Hooks (4タイプ), Permission Modes | キッチンタイマー・衛生管理規則 | 運営管理 |
| **6. 配布** | Plugins | フランチャイズキット | チームへ展開する |

以降、各レイヤーを順に解説します。

---

## 1. コア：エージェントループ（厨房の心臓部）

すべての拡張機能はここに接続します。Claude Codeの根幹であるエージェントループ、つまり「考える→道具を使う→結果を見る→次を考える」の繰り返しです。

### 料理長 ＝ メインエージェント

厨房の最高責任者です。お客様（ユーザー）からオーダーを受け取り、自分で調理するか、部下に振るかを判断します。簡単な一品なら自分でサッと作りますが、コース料理のような複雑なタスクでは、レイヤー2のエージェントたちに委任します。

### 調理器具 ＝ ビルトインTools

料理長と全スタッフが使う基本的な調理器具です。Claude Codeに最初から組み込まれており、拡張機能ではありません。

| 調理器具 | Tool名 | 用途 |
|---|---|---|
| 包丁 | Write / Edit | ファイルの作成・編集 |
| ルーペ | Read | ファイルの読み取り |
| シノワ(ふるい) | Grep / Glob | 検索・ファイル探索 |
| 万能鍋 | Bash | シェルコマンド実行 |
| 電話 | WebFetch / WebSearch | Web情報の取得 |
| 人材派遣 | Agent | サブエージェント起動（旧称: Task） |

:::message
**Task → Agent へのリネーム**
Claude Code v2.1.63 以降、サブエージェント起動ツールは `Task` から `Agent` に改名されました。既存の `Task(...)` 表記もエイリアスとして引き続き動作します。
出典: [Restrict which subagents can be spawned](https://code.claude.com/docs/en/sub-agents#restrict-which-subagents-can-be-spawned)
:::

### 作業台と整理 ＝ Context Window / Compaction

厨房の作業台には限りがあります。食材（ファイルの内容）、道具の出力（ツールの結果）、メモ（会話履歴）を載せすぎると溢れます。

**Compaction（まな板の整理）** は、作業台が溢れた時に不要な情報を片付けて重要な情報だけ残す整理作業です。長時間のセッションでは自動的に発動します。直近で使ったSkillsは再アタッチされますが、古いものは落ちることがあります。

:::message
**なぜコアを独立層にするのか？**
Main Agent・Tools・Context Windowは他の全機能の**前提条件**です。Skills も Subagents も Hooks も、すべてこのエージェントループの上で動きます。ここを理解せずに拡張機能だけ見ても、全体像はつかめません。
:::

---

## 2. エージェント：委任と並列化（料理人の配置）

料理長ひとりでは大量のオーダーを捌けません。このレイヤーは**仕事を他のエージェントに委任する仕組み**です。ビルトイン → カスタム → Teams と段階的にスケールアウトします。

### ビルトイン副料理長チーム ＝ ビルトインサブエージェント

最初からお店に所属している直属の部下です。それぞれ専門領域が異なります。

| 厨房での役割 | サブエージェント | 特徴 |
|---|---|---|
| **偵察係** | Explore | 読み取り専用。冷蔵庫に何があるか見に行くだけ。安価なHaikuモデルで動作 |
| **スーシェフ（副料理長）** | Plan | コンテキストを収集し、作戦を立てる。変更は加えない。プランモード時に自動起動 |
| **何でも屋** | general-purpose | 全ツール利用可。探索とコード修正の両方が必要な複雑タスク向け |
| **(裏方ヘルパー)** | statusline-setup / Claude Code Guide | `/statusline`設定や Claude Code 自体の質問に自動起動。ユーザが直接呼ぶことはほぼない |

各サブエージェントは**専用の作業台（コンテキストウィンドウ）を持ちます**。偵察係が大量のファイル情報を調べても、料理長の作業台には結果の要約だけが戻ってきます。これがコンテキスト汚染を防ぐ最大のメリットです。

出典: [Built-in subagents](https://code.claude.com/docs/en/sub-agents#built-in-subagents)

### 専門料理人（自作）＝ カスタムサブエージェント

パティシエ（デザート専門）、ロティスール（焼き場専門）のように、自分のお店のメニューに合わせて雇う専門スタッフです。

`.claude/agents/` にマークダウンファイルを置いて定義します。アプリケーションのコンポーネント別にレビュアーを分ける例:

```yaml:.claude/agents/backend-reviewer.md
---
name: backend-reviewer
description: アプリケーションのbackendロジックのコードを専門にレビューする。*.pyなどの変更があるMRでproactiveに使う。
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
skills:
    - python-best-practices        # SOLID原則、Factory Method, Abstract Factory, ...
    - security-checklist        # 認証認可、脆弱ライブラリ検知
    - client-ccoe-requirements        # クライアント固有の統制要件
model: opus
# 追加で指定できる主なフィールド（公式16項目から抜粋）:
# permissionMode: plan        # このエージェント単体でプランモード強制
# isolation: worktree         # git worktree 内で動かす（並列実行で衝突防止）
# memory: project             # .claude/agent-memory/<name>/ に知見を蓄積
# maxTurns: 20                # 暴走防止
# background: true            # メインとは独立して裏で走らせる
# mcpServers: [...]           # このサブエージェント専用MCPを付与
---

あなたはシニアCloud Application Engineerとして Python Backendロジックのコードレビューを行います。

## レビュー観点
プリロードされた skills のチェックリストを全て適用してください。
言語横断の命名規約・コミット規約は CLAUDE.md および .claude/rules/ を参照。

## 判定
CRITICAL / HIGH / MEDIUM / LOW / INFO の5段階で分類。
```

`tools`フィールドで**使える調理器具を制限できます**。「このレビュー担当者には包丁（Write）は持たせない。読むだけ」という安全設計が可能です。

:::message
**サブエージェントを別の副料理人として隔離したいとき** → `isolation: worktree`
並列レビューで複数のレビュアーを同時起動するとき、worktreeで独立コピーを持たせればファイル書込みが衝突しません。レビュー後に変更がなければ自動クリーンアップされます。
:::

:::message
**サブエージェントが学習する「専門料理人の個人ノート」** → `memory: project`
`memory` を有効化すると `.claude/agent-memory/<name>/` に永続ディレクトリが与えられ、Read/Write/Editツールが自動で許可されます。セッションを跨いでノウハウが蓄積されるため、「よくあるアンチパターン集」のようなものを育てられます。
:::

#### サブエージェントの起動を制御する: `Agent(subagent-name)` 構文

`--agent` で起動する**メインスレッドのエージェント**が、どのサブエージェントを呼び出せるかをallowlist制御できます:

```yaml
---
name: coordinator
tools: Agent(worker, researcher), Read, Bash   # workerとresearcherのみ起動可
---
```

または deny リスト形式でブロック:

```json:.claude/settings.json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```

出典: [Supported frontmatter fields](https://code.claude.com/docs/en/sub-agents#supported-frontmatter-fields), [Restrict which subagents can be spawned](https://code.claude.com/docs/en/sub-agents#restrict-which-subagents-can-be-spawned)

### 合同バンケット班 ＝ Agent Teams

200名の披露宴のような大型案件では、一つの厨房では回りきりません。Agent Teamsは**複数の独立した厨房（Claude Codeセッション）を立ち上げ、チームリーダーの下で協調させる仕組み**です。

```
# 自然言語で指示するだけでチームが立ち上がる
認証モジュールのリファクタリングを並列で進めたい。
エージェントチームを作って：
- 1人はJWT処理の分離担当
- 1人はセッション管理の分離担当
- 1人はテスト作成担当
```

:::message alert
Agent Teamsは実験的機能です（v2.1.32以降）。有効化方法は[付録](#付録：実験的機能の具体的設定リファレンス)を参照してください。
:::

### コラム：Subagent vs Agent Teams — 個室の専門担当と合同厨房の違い

| | サブエージェント | Agent Teams |
|---|---|---|
| 厨房の例え | **個室の専門担当**（報告は料理長のみ） | **複数の副料理長がフロアで会話しながら協業** |
| コミュニケーション | 料理長に結果を返すのみ | メンバー同士が直接メッセージ送信 |
| タスク管理 | 料理長が個別に指示 | 共有タスクリストから自分で取る |
| コンテキスト | 親セッション内に作られる | 完全に独立したセッション |
| コスト | 比較的低い | 各メンバーが独立セッション＝トークン消費大 |
| 使い所 | 結果だけ返ればよい集中タスク | 議論・共有・協調が必要な複雑作業 |

**判断基準：** 作業者同士が会話する必要があるか？ Noならサブエージェント、Yesなら Agent Teams。

出典: [Compare with subagents](https://code.claude.com/docs/en/agent-teams#compare-with-subagents)

---

## 2〜3 レイヤーの橋渡し: Subagent × Skills × Rules の三層構造

Subagent（専門担当者）と Skills（レシピ）と CLAUDE.md / `.claude/rules/`（ハウスルール）は、**単独で使うのではなく組み合わせる**のが Claude Code の流儀です。言語別のコードレビューを例にすると:

```
Subagent: backend-reviewer          ← 専門担当者の人格・モデル・ツール制限
  ├── tools: Read, Grep, Glob, Bash  ← この担当者に持たせる道具
  ├── skills:                         ← 担当者にプリロードする専門レシピ
  │     - python-best-practices    
  │     - security-checklist    ← 他のSubagentとも共有可能
  │     - client-ccoe-requirements    
  └── (暗黙)                          ← CLAUDE.md + .claude/rules/*.md は全員共通で参照
        ├── CLAUDE.md                 ← 店のハウスルール（常駐）
        └── .claude/rules/            ← 命名規約・コミット規約など、スキル横断の共通ルール
                ├── naming.md
                └── commit-message.md
```

### 責務分担の判断基準

| 種別 | 置き場所 | こう書く | 例 |
|---|---|---|---|
| **Subagent** | `.claude/agents/*.md` | 「誰が」作業するか | `terraform-reviewer`, `python-reviewer`, `security-reviewer` |
| **Skills** (スキル固有) | `.claude/skills/*/SKILL.md` | 「特定観点の」作業手順・チェックリスト | `terraform-best-practices`, `api-conventions`, `error-handling-patterns` |
| **rules** (共通) | `.claude/rules/*.md` or `CLAUDE.md` | 「スキル横断で」常に守るルール | `naming.md`, `commit-message.md`, `design-doc-traceability.md` |
| **Slash Command** (= Skillの一種) | `.claude/skills/<name>/SKILL.md` + `/<name>` 起動 | 「一連の作業フロー」 | `/review-mr`（検出→配線→集約→記録） |

### 厨房での例え（整理版）

| 構成要素 | 厨房の比喩 |
|---|---|
| Subagent | 調理担当者（和食料理人／フレンチ料理人／パティシエ） |
| Skills | その担当者が持参する専門レシピ集（和食なら「出汁の引き方」「盛付けの決まり」） |
| rules / CLAUDE.md | 店全体で掲示されているハウスルール（衛生基準・営業時間・食材調達ルール） |
| Slash Command | 壁掛けの日替わりメニュー札（押すと「前菜→主菜→デザート」の一連のフロー起動） |

:::message
**鉄則**: Subagent は「誰が」、Skills は「何の観点で」、rules は「全員共通で守るべき」ものです。
**スキル横断の汎用ルールを特定Skillに埋め込まない** — 他のSkillから参照できなくなります。
複数サブエージェントで共有したい観点（`security-owasp-checklist` 等）は**単一のSkillとして定義**して、各Subagentの `skills:` から参照するのが公式推奨パターンです。
:::

出典: [Preload skills into subagents](https://code.claude.com/docs/en/sub-agents#preload-skills-into-subagents)

---

## 3. 知識とコンテキスト：Claudeが知っていること（レシピ棚）

料理人の腕（エージェント）だけでなく、**何を知っているか**がアウトプットの質を決めます。このレイヤーの機能は**ロードタイミング**で明確に使い分けます。

### ハウスルール ＝ CLAUDE.md（常時ロード）

「うちの店はソースに砂糖を使わない」「盛り付けは必ず白い皿」のような、**変わらない店のルール**です。セッション開始時に常に読み込まれます。

```markdown:CLAUDE.md
## プロジェクト規約
- TypeScript strict mode必須
- 関数は200行以内
- テストはJest + React Testing Library、カバレッジ80%以上
- コミット前にnpm testを実行すること

## Agent Team Rules

### ファイル所有権（One File, One Owner）

- 各 teammate は **リードが割り当てたディレクトリのみ** に書き込む
- 読み取りは全ファイル可能だが、書き込みは所有ディレクトリに限定する
```

:::message
**CLAUDE.mdはコンパクトに保つ。** 肥大化したら、特定作業時に適用のルールはSkillsに、複数作業で常時適用のルールは`.claude/rules/`に分離しましょう。
`.claude/rules/*.md` は Hook イベント `InstructionsLoaded` で自動ロードされ、CLAUDE.md と同様にセッションコンテキストに取り込まれます。
出典: [InstructionsLoaded hook](https://code.claude.com/docs/en/hooks#instructionsloaded)
:::

### レシピブック ＝ Skills（オンデマンド）

分厚いレシピ本が棚に並んでいるイメージ。料理長は**目次（name と description）だけを常に把握**しており、「今日はフレンチだからこの章を開こう」と必要な時だけ詳細を読みます。これが**プログレッシブ・ディスクロージャー**と呼ばれる設計です。

Skills は公式的に2種類の使い方があります:

| 種別 | 役割 | 代表的な frontmatter |
|---|---|---|
| **Reference content** | 規約・パターン・スタイルガイド・ドメイン知識。Claudeが会話中に参照 | 素の `name` + `description` |
| **Task content** | 「この手順を実行せよ」という一連のワークフロー。`/name` で手動起動想定 | `disable-model-invocation: true` で自動起動を防ぐ |

言語別レビュー例だと、`terraform-best-practices`（Reference）はサブエージェントがプリロードして参照、`/review-mr`（Task）はユーザが明示的に起動、という使い分けになります。

```yaml:.claude/skills/terraform-best-practices/SKILL.md
---
name: terraform-best-practices
description: Terraform/TerragruntのIaC設計ベストプラクティス。レイヤ分離、tfstate管理、moduleパターン、closed-network対応を含む。
allowed-tools: 
    - Read 
    - Grep 
    - Glob
---

# Terraform ベストプラクティス（レビュー観点）

## 1. レイヤ分離
- providers / networking / security / data / application の5レイヤ分離を確認
- tfstate は環境×レイヤ単位で分割（crossover した同一backendは指摘）

## 2. Terragrunt DRYパターン
- common.hcl / env.hcl / layer.hcl の三層構造
- include.path の適切な利用
...
```

:::message
**書式注意**: `allowed-tools` は **スペース区切り または YAMLリスト**です。カンマ区切りは公式仕様外です。
出典: [Skills - Frontmatter reference](https://code.claude.com/docs/en/skills#frontmatter-reference)
:::

Skillsは最も汎用的な拡張メカニズムで、Claude Code・claude.ai・APIのすべてで動作します。サブエージェントの中でも使えます（「パティシエにデザートのレシピ本を渡す」）。

### 壁掛けメニュー札 ＝ SkillsによるSlash Command（手動トリガー）

:::message alert
**重要**: `.claude/commands/` は Skills に統合されました（2026年4月時点）。既存の `.claude/commands/*.md` も動作しますが、**新規作成は Skills で行うことが公式推奨**です。Skills は以下が追加できる上位互換です:
- 補助ファイル（SKILL.md以外のテンプレート・スクリプト・リファレンス）
- 自動起動の制御（`disable-model-invocation`, `user-invocable`）
- サブエージェント内での実行（`context: fork`）

公式引用: "Custom commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way."
出典: [Extend Claude with skills](https://code.claude.com/docs/en/skills)
:::

よく出るワークフローは **Task content 型の Skill** として定義し、`/` でユーザが手動起動します。「各観点でのレビュー→レビュー記録→指摘検討→横展開→指摘反映」のような一連のフローがこの典型例です:

```yaml:.claude/skills/review-mr/SKILL.md
---
name: review-mr
description: MRレビューの一連のフロー。対象MR特定→言語別レビュアー起動→指摘集約→レビュー記録→横展開候補抽出
disable-model-invocation: true    # /review-mr で明示起動のみ
argument-hint: "[MR番号 or ブランチ名]"
allowed-tools: 
    - Bash(gh *) 
    - Read 
    - Grep
---

# MR レビュー実行フロー

## Step 1: 対象MR特定
!`gh pr view $ARGUMENTS --json files,title,body`

## Step 2: 言語別レビュアーへ配線
- `*.tf` / `*.hcl` があれば @terraform-reviewer を @-mention で起動
- `*.py` があれば @python-reviewer を起動
- `*.tsx` / `*.ts` があれば @react-reviewer を起動

## Step 3: 指摘の集約・レビュー記録
...

## Step 4: 横展開候補の抽出
同一のアンチパターンが他のファイルで発生していないか @terraform-reviewer に再調査を依頼
...
```

:::message
**Skillでshellコマンドが `` !`...` `` で埋め込めます**（Dynamic context injection）。
上の例では `gh pr view` の結果が Claude に渡される**前に**展開されるので、Claude は実データを見た状態でフローに入れます。
出典: [Inject dynamic context](https://code.claude.com/docs/en/skills#inject-dynamic-context)
:::

### 常連ノート ＝ Memory（セッション横断）

「田中さんはパクチーNG、辛さ控えめ」を記録しておく常連ノートです。セッションをまたいで保持されます。サブエージェントにも`memory`フィールドで永続ディレクトリを与えることができます（`.claude/agent-memory/<name>/`）。

### コラム：何をどこに書くか — 三層構造での使い分け

| 判断基準 | 置き場所 |
|---|---|
| Claudeが**常に**知っておくべき店のルール | **CLAUDE.md** |
| スキル・サブエージェント**横断で共通**のルール（命名、コミット規約、設計書トレーサビリティ） | **`.claude/rules/*.md`** |
| **特定観点・特定サブエージェント**で使う専門知識・チェックリスト（Reference content型 Skills） | **`.claude/skills/<name>/SKILL.md`** |
| **毎回同じプロンプト**を打っている定型作業フロー（Task content型 Skills、`/name` で起動） | **`.claude/skills/<name>/SKILL.md`**（`disable-model-invocation: true`付き）|
| 複数サブエージェントで**共有したい観点**（`security-owasp-checklist`など） | **単一のSkillとして定義**し、各Subagentの `skills:` から参照 |
| Claudeが規約を**2回間違えた** | **CLAUDE.md** または **`.claude/rules/`** に追記 |
| **同じ手順書を3回**チャットに貼り付けた | **Skills**に切り出し |
| **外部データ**をブラウザからコピペしている | Skillsではなく**MCP**（レイヤー4） |
| **毎回必ず**実行してほしい処理 | Skillsではなく**Hooks**（レイヤー5） |

---

## 4. 外部接続：厨房の外とつながる（仕入れルート）

### 仕入れ業者 ＝ MCP（Model Context Protocol）

MCPはClaude Codeを外部システムに接続するための**オープンスタンダード**です。厨房の中にある道具（Tools）とは異なり、**厨房の外にいる仕入れ業者**に電話して食材を届けてもらうイメージです。

| 厨房での例え | MCP接続先 | 何を仕入れるか |
|---|---|---|
| 築地の魚市場 | GitHub | ソースコード |
| 冷蔵倉庫 | PostgreSQL / Supabase | データベース |
| 酒屋 | Slack | チームの会話 |
| 食材カタログ | Web Search | 最新情報 |
| 書類配送 | Google Drive | ドキュメント |

```json:.mcp.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token>" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://localhost:5432/mydb"]
    }
  }
}
```

### コンテキスト節約のコツ — メインから見えないMCPを作る

MCPサーバを `.mcp.json` でグローバルに登録すると、全ツール定義がメイン会話のコンテキストに常駐します。特定のサブエージェントでしか使わないMCPは、**サブエージェントの frontmatter の `mcpServers:` にインライン定義**すると、そのサブエージェント起動時だけ接続され、メインのコンテキストを圧迫しません。

```yaml:.claude/agents/browser-tester.md
---
name: browser-tester
description: Playwrightで実ブラウザでの動作確認を行う
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github          # 既設定のgithub MCPを参照
---

Playwright ツールでナビゲート、スクリーンショット、ページ操作を行ってください。
```

出典: [Scope MCP servers to a subagent](https://code.claude.com/docs/en/sub-agents#scope-mcp-servers-to-a-subagent)

---

## 5. 自動化と安全：厨房の運営管理（タイマーと衛生管理）

このレイヤーの機能は**エージェントループの外で動きます**。LLMの判断に依存しない確定的な処理と安全制御です。

### キッチンタイマーとその仲間 ＝ Hooks

LLMは同じ指示を受けても毎回微妙に違う動作をする可能性があります。しかし「ファイル保存時に必ずlintを走らせる」のような処理は、100%確実に実行されなければなりません。**Hooksはそれを保証します。**

#### Hookは2段構造で理解する（読者が混乱しやすいポイント）

| レイヤー | 役割 | 置き場所 |
|---|---|---|
| **① 登録**（レシピへの付箋） | どのイベントで何を起動するか宣言 | `settings.json`（user/project/local）/ Plugin の `hooks/hooks.json` / Subagent・Skill frontmatter |
| **② ハンドラ本体**（実際のタイマー） | 実行されるスクリプト・HTTPエンドポイント・プロンプト・サブエージェント | 任意の場所。慣例として `.claude/hooks/*.sh` (`$CLAUDE_PROJECT_DIR` 基準) |

:::message alert
**`.claude/hooks/` にスクリプトを置くだけでは動きません。**
必ず `settings.json` 側で `PreToolUse` 等のイベントに紐付ける登録が必要です。
出典: [Hook locations](https://code.claude.com/docs/en/hooks#hook-locations), [Reference scripts by path](https://code.claude.com/docs/en/hooks#reference-scripts-by-path)
:::

#### Hookの4タイプ — キッチンタイマー以外にもバリエーションあり

| タイプ | 厨房での例え | 用途 |
|---|---|---|
| `command` | キッチンタイマー | シェルスクリプト実行（最も一般的） |
| `http` | 外部の検品業者に電話 | HTTPエンドポイントにPOST |
| `prompt` | セカンドオピニオン (LLMによる判定) | Claudeモデルに単発プロンプトで yes/no 判定 |
| `agent` | 抜き打ち検品担当の副料理長 | サブエージェントを起動して検証（Read/Grep等ツール使用可） |

出典: [Hook handler fields](https://code.claude.com/docs/en/hooks#hook-handler-fields)

#### 代表的なHookイベント

Claude Codeは**30近いHookイベント**を提供しています。ここでは代表的なものに絞って紹介します（全イベントは[公式リファレンス](https://code.claude.com/docs/en/hooks#hook-events)を参照）。

| タイミング | イベント | 厨房での例え |
|---|---|---|
| 開店 | `SessionStart` | 朝礼で今日の仕入れ情報を共有 |
| ハウスルール読込 | `InstructionsLoaded` | 店の掲示板の最新版を確認（CLAUDE.md / .claude/rules/ が読まれた瞬間）|
| 注文受付 | `UserPromptSubmit` | オーダー票の書式チェック |
| 調理前 | `PreToolUse` | 食材の安全検査（危険なコマンドをブロック） |
| 許可ダイアログ表示時 | `PermissionRequest` | 食材出庫の承認印を押す |
| auto modeの拒否 | `PermissionDenied` | 自動検品が食材をNGにしたとき |
| 調理後 | `PostToolUse` | 盛り付け検品（自動フォーマット） |
| 調理失敗時 | `PostToolUseFailure` | 焦げた料理の後始末 |
| サブエージェント起動/終了 | `SubagentStart` / `SubagentStop` | 副料理人の出勤/退勤 |
| タスクのライフサイクル | `TaskCreated` / `TaskCompleted` | オーダー票の発行/完了印 |
| Teammateアイドル | `TeammateIdle` | 仲間が手を止めようとした瞬間（品質ゲート発動）|
| 設定変更 | `ConfigChange` | 厨房マニュアルの変更監査 |
| ディレクトリ変更 | `CwdChanged` | 担当フロアの変更 |
| ファイル変更 | `FileChanged` | 監視対象食材の入替 |
| コンパクション | `PreCompact` / `PostCompact` | まな板の整理前後 |
| MCP elicitation | `Elicitation` / `ElicitationResult` | 仕入れ業者からの追加確認 |
| APIエラーによる停止 | `StopFailure` | 冷蔵庫故障で業務中断 |
| 提供完了 | `Stop` | 全品出し終わった後のテーブルチェック |
| 閉店 | `SessionEnd` | 翌日への申し送り |

```json:.claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/format.sh"
          }
        ]
      }
    ]
  }
}
```

:::message
**よくある誤解：Hooks ≠ 調理器具**
調理器具（包丁・鍋）に相当するのはTools（Read, Write, Bash等）です。Hooksは「特定のタイミングで確定的に発火する自動処理」なので、**キッチンタイマー・抜き打ち検品・外部検品業者への電話**といった「料理の流れに割り込む仕組み」が正しいアナロジーです。
:::

### 衛生管理規則 ＝ Permission Modes

食品衛生法に相当する安全装置です。「誰が何を触れるか」を制御します。

| モード | 厨房での例え | 説明 |
|---|---|---|
| **default** | 毎回料理長が確認 | 操作ごとにユーザーに許可を求める |
| **acceptEdits** | 「包丁仕事は任せた」 | ファイル編集は自動許可、コマンド実行は確認 |
| **plan** | 「まず献立だけ見せて」 | 読み取りのみ、変更は不可 |
| **auto** | バックグラウンドの品質管理官 | バックグラウンド分類器モデルがコマンドと保護ディレクトリ書込みをレビュー |
| **dontAsk** | 「断りなく許可は出さない」 | 許可プロンプトを自動拒否（明示的に`allow`されたツールのみ実行可） |
| **bypassPermissions** | 「好きにやれ」 | 全操作を無条件許可（非推奨） |

`allow` / `deny`リストで調理器具の使用許可を細かく制御できます。Hooksの`PreToolUse`と組み合わせることで、より堅牢な防御層（Defense in Depth）を構築できます。

出典: [Permission modes (subagents)](https://code.claude.com/docs/en/sub-agents#permission-modes)

---

## 6. 配布：チームへ展開する（フランチャイズキット）

### フランチャイズキット ＝ Plugins

新規出店する時に「うちのレシピ、ルール、仕入れ先、スタッフ構成、タイマー設定」を全部まとめたキットを渡す——それがPluginsです。

**レイヤー2〜5（Agents + Skills + Commands + Hooks + MCP）を任意の組み合わせで一括パッケージ化**し、チームに配布できます。

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json       # プラグインのメタ情報
├── commands/              # Slash Commands（レガシー。新規はskills推奨）
├── agents/                # サブエージェント定義
├── skills/                # Skills（Reference/Task両方）
├── hooks/                 # Hooks（hooks.jsonで登録）
├── .mcp.json              # MCP接続設定
└── README.md
```

```bash
# 公式プラグインのインストール
/plugin add anthropics/agent-sdk
```

**初めてClaude Codeを拡張するなら、まず公式プラグインを入れてみるのが手軽**です。実績のある構成が一式手に入ります。

---

## 実践ガイド：何から始めるか

公式ドキュメントの「[Build your setup over time](https://code.claude.com/docs/en/features-overview#build-your-setup-over-time)」に基づく段階的な導入フローです。

| ステップ | きっかけ | やること | レイヤー |
|---|---|---|---|
| 1 | Claudeが規約を間違えた | **CLAUDE.md**にルールを追記 | 3. 知識 |
| 2 | 複数の作業で同じルールが必要 | **`.claude/rules/*.md`** に切り出し | 3. 知識 |
| 3 | 同じプロンプトを繰り返し打っている | **Skills** に切り出す（`/name` で起動可能）| 3. 知識 |
| 4 | ブラウザからデータをコピペしている | **MCP**で外部サービスを接続 | 4. 外部接続 |
| 5 | サイドタスクがコンテキストを圧迫 | **サブエージェント**に委任（`skills:`で観点をプリロード）| 2. エージェント |
| 6 | 毎回必ず実行したい処理がある | **Hooks**で自動化 | 5. 自動化 |
| 7 | 他のリポジトリでも同じ構成を使いたい | **Plugins**にパッケージ化 | 6. 配布 |
| 8 | 並列探索・大規模リファクタが必要 | **Agent Teams**で複数セッション | 2. エージェント |

最初から全部揃える必要はありません。**レイヤー3（知識）→ 4（外部）→ 2（エージェント）→ 5（自動化）→ 6（配布）** の順で必要になった時に追加していくのが自然な流れです。

---

## 付録：実験的機能の具体的設定リファレンス

### Agent Teams（合同バンケット班）の開店準備

有効化は`~/.claude/settings.json`に1行追加するだけです。

```json:~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

:::message
**前提条件**
- Claude Code v2.1.32以降（`claude --version`で確認）
- モデルは task に応じて指定（公式ドキュメントで特定モデルの推奨は明示されていないが、複雑な設計タスクは Opus 系を明示的に指示すると良い）
- Pro ($20/月) またはMax ($100〜$200/月) プラン
:::

**tmuxによる分割表示の設定（推奨）**

デフォルトは `"auto"` で、tmuxセッション内にいる場合はsplit-paneモード、それ以外はin-processモードで起動します。

```bash
tmux new-session -s dev
claude
```

| 表示モード | 説明 | 環境 |
|---|---|---|
| **in-process** | 同一ターミナル内。`Shift+Down`で切り替え | デフォルト(tmux外) |
| **split-pane** | 各チームメイトが独立ペイン | tmux / iTerm2 |
| **auto** | 環境を自動判定 | デフォルト設定値 |

**サブエージェント定義との組み合わせ**

繰り返し使うチーム構成は、`.claude/agents/`にサブエージェント定義を作っておくと再利用できます。

```yaml:.claude/agents/security-reviewer.md
---
name: security-reviewer
description: セキュリティ観点のコードレビュー担当
tools: Read, Grep, Glob, Bash
model: opus
---

セキュリティレビューの専門家として行動してください。
```

```
# Agent Teams起動時にサブエージェント名を指定
PRのレビューチームを作って。
- security-reviewer をセキュリティ担当に
- もう1人をパフォーマンス担当に
- もう1人をテストカバレッジ担当に
```

:::message alert
**Agent Teamsの既知の制限事項（2026年4月時点）**
- `/resume`でチームメイトは復元されない（新規スポーンが必要）
- タスク完了ステータスが遅延することがある
- 1セッション1チームまで
- チームメイトがさらにチームをネストすることは不可
出典: [Agent Teams - Limitations](https://code.claude.com/docs/en/agent-teams#limitations)
:::

---

### Hooks（キッチンタイマー）の設置方法

`~/.claude/settings.json`（全プロジェクト共通）または`.claude/settings.json`（プロジェクト単位）に記述します。

**実用例1：ファイル編集後の自動フォーマット**

```json:.claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/format.sh"
          }
        ]
      }
    ]
  }
}
```

**実用例2：危険なコマンドのブロック**

```json:.claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/pre-bash-firewall.sh"
          }
        ]
      }
    ]
  }
}
```

**実用例3：セッション開始時のコンテキスト注入**

```json:.claude/settings.json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"現在のブランチ: '$(git branch --show-current)'\"}}'"
          }
        ]
      }
    ]
  }
}
```

**実用例4：セッション開始時の環境変数セットアップ（`CLAUDE_ENV_FILE`）**

SessionStart / CwdChanged / FileChanged Hookは、`CLAUDE_ENV_FILE` 環境変数経由で、**セッション全体の Bash 実行に持続する環境変数**をセットできます。

```bash:.claude/hooks/setup-env.sh
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export AWS_PROFILE=dev' >> "$CLAUDE_ENV_FILE"
  echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

これで、以降 Claude が実行する全 Bash コマンドにこれらの変数が適用されます。direnv や nvm のような per-directory 環境管理と相性が良いのがこの機能です。

出典: [Persist environment variables](https://code.claude.com/docs/en/hooks#persist-environment-variables)

:::details Hookの終了コードの意味
| 終了コード | 意味 | 使い所 |
|---|---|---|
| `0` | 正常終了（許可） | 通常の処理 |
| `2` | ブロック（`PreToolUse`, `UserPromptSubmit`, `Stop` など） | 危険な操作の阻止 |
| その他の非ゼロ | 非ブロッキングエラー | ユーザーへの警告表示 |

イベントごとの `exit 2` の挙動詳細は [Exit code 2 behavior per event](https://code.claude.com/docs/en/hooks#exit-code-2-behavior-per-event) を参照。
:::

---

### Permission Modes（衛生管理規則）の設定

```json:~/.claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm test:*)",
      "Bash(npm run lint:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(git push --force:*)"
    ]
  }
}
```

---

### 設定ファイルの配置場所まとめ

| ファイル | 配置場所 | スコープ | 内容 |
|---|---|---|---|
| `~/.claude/settings.json` | ホーム | 全プロジェクト共通 | Hooks, Permissions, 環境変数 |
| `.claude/settings.json` | プロジェクトルート | プロジェクト単位 | Hooks, Permissions（プロジェクト固有） |
| `.claude/settings.local.json` | プロジェクトルート | 個人用（gitignore推奨） | 個人固有の設定 |
| `CLAUDE.md` | プロジェクトルート | プロジェクト単位 | ハウスルール、規約 |
| `~/.claude/CLAUDE.md` | ホーム | 全プロジェクト共通 | 個人の共通ルール |
| `.claude/rules/*.md` | プロジェクトルート | プロジェクト単位 | スキル・サブエージェント横断の共通ルール（`InstructionsLoaded`Hookで常時ロード）|
| `.claude/skills/<name>/SKILL.md` | プロジェクト or ホーム | 配置場所による | Skills定義（Reference/Task 両方） |
| `.claude/agents/*.md` | プロジェクト or ホーム | 配置場所による | サブエージェント定義 |
| `.claude/agent-memory/<name>/` | プロジェクト or ホーム | 配置場所による | サブエージェントの永続メモリ（`memory` フィールド有効時） |
| `.claude/commands/*.md` | プロジェクト or ホーム | 配置場所による | **(レガシー)** Slash Commands。Skillsに統合済 — 新規は `.claude/skills/` 推奨 |
| `.mcp.json` | プロジェクトルート | プロジェクト単位 | MCP接続設定 |
| `.claude/hooks/*.sh` | プロジェクトルート | プロジェクト単位 | Hookハンドラスクリプト本体（`settings.json`から参照）|

:::message
**`.claude/commands/` は Skills にマージ済み**です。既存の command ファイルはそのまま動きますが、新規は Skills で作成すると補助ファイル・`disable-model-invocation`・`context: fork` など上位互換の機能が使えます。
Skills は `/<skill-name>` で直接呼び出し可能です。
:::

---

## おわりに

Claude Codeの機能は多岐にわたりますが、**6つのレイヤー**で整理すると構造がすっきりします。

1. **コア** — 料理長 + 調理器具 + 作業台（全機能の土台）
2. **エージェント** — 副料理長 → 専門料理人 → バンケット班（段階的スケールアウト）
3. **知識** — ハウスルール → 共通規約 → 専門レシピ → メニュー札 → 常連ノート（ロードタイミングで使い分け）
4. **外部接続** — 仕入れ業者（厨房の外とつながる）
5. **自動化と安全** — キッチンタイマー4種 + 衛生管理（ループの外で確実に動く）
6. **配布** — フランチャイズキット（上記をパッケージして共有）

そして特に重要なのは、**Subagent × Skills × rules の三層構造**です:

- **Subagent** が「誰が」作業するか（専門担当者の人格・ツール制限）
- **Skills** が「何の観点で」作業するか（チェックリスト・手順書。`skills:`でSubagentにプリロード）
- **rules / CLAUDE.md** が「全員共通で守るべき」ルール（命名・コミット規約など）

厨房運営と同じで、最初から全スタッフ・全設備を揃える必要はありません。**レイヤー3の知識整備から始めて、必要になった時にレイヤーを追加していく**のが王道です。

---
## 参考リンク

- [Extend Claude Code（公式 — 機能比較の決定版）](https://code.claude.com/docs/en/features-overview)
- [How Claude Code works（公式 — エージェントループの解説）](https://code.claude.com/docs/en/how-claude-code-works)
- [Claude Code サブエージェント公式ドキュメント](https://code.claude.com/docs/en/sub-agents)
- [Agent Teams 公式ドキュメント](https://code.claude.com/docs/en/agent-teams)
- [Skills 公式ドキュメント](https://code.claude.com/docs/en/skills)
- [Skills 公式ブログ](https://claude.com/blog/skills-explained)
- [Agent Skills エンジニアリングブログ](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Hooks リファレンス公式ドキュメント](https://code.claude.com/docs/en/hooks)
- [Plugins 公式ブログ](https://claude.com/blog/claude-code-plugins)
- [MCP 公式ドキュメント](https://code.claude.com/docs/en/mcp)
- [Claude Code Full Stack解説 (alexop.dev)](https://alexop.dev/posts/understanding-claude-code-full-stack/)
