---
title: "Claude Certified Architect – Foundations（CCAF）に合格した件"
emoji: "🏆"
type: "idea"
topics: ["claude", "anthropic", "claudecode", "certification", "ccaf"]
published: true
---
## はじめに

2026年5月31日、[**Claude Certified Architect – Foundations（CCAF）**](https://claudecertifications.com/claude-certified-architect) を受験し、757点（合格ライン720点）で合格しました。

本記事では、勉強法・試験傾向・判断基準や受験後の結果通知までの話など、CCAFを受験しようか迷っている方の参考になりそうな情報をまとめます。

---

## CCAFとは

CCAFは**Anthropicが提供するClaude公式認定試験**です。Claude APIやClaude Codeを使ったシステム設計の知識を問います。

| 項目       | 内容            |
| ---------- | --------------- |
| 問題数     | 60問            |
| 試験時間   | 120分           |
| スコア範囲 | 100〜1000       |
| 合格ライン | 720             |
| 形式       | 多肢選択（CBT） |

試験は5つのドメインで構成されています：

1. **Agentic Architecture & Orchestration** — マルチエージェント設計・オーケストレーション
2. **Tool Design & MCP Integration** — ツール設計・MCPサーバー統合
3. **Claude Code Configuration & Workflows** — Claude Code設定・ワークフロー
4. **Prompt Engineering & Structured Output** — プロンプト設計・構造化出力
5. **Context Management & Reliability** — コンテキスト管理・信頼性

> 合格後はAnthropicのAcademyプロフィールからデジタルバッジを共有できます。

---

## 筆者プロフィール

参考のために、自分のバックグラウンドを簡単に。

- **TOEIC 900点** — 英語の選択肢をそのまま読んで判断できる程度
- **Claude Codeを日常的に開発で使用** — 自作スキルや設定のメンテもしている
- **Anthropic Partner Bootcamp参加済み** — エージェント設計の基礎知識あり

---

## 勉強法

### 1. Anthropicコース「[Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api)」受講

まずAnthropicが提供する公式の学習コースを受講しました。試験ガイドのドメインと対応したトピックをカバーしており、体系的に知識を整理できます。

### 2. 公式Practice ExamをClaudeで徹底解析（★最重要）

**これが一番効いた勉強法です。**

[公式のPractice Exam](https://anthropic-partners.skilljar.com/anthropic-certification-practice-exam/425721/scorm/17p1a5iqsma8x)を解いたあと、各問題についてClaudeに以下を聞きました：

1. **なぜこの選択肢が正解か**
2. **なぜ他の選択肢は不正解か**
3. **どのような状況なら、不正解の選択肢が正解になるか**

特に3つ目が重要です。「A＞Bだからこの問題はA」というだけでなく、「どんな文脈ならBが正解か」まで理解すると、本番で似た問題が出ても動じなくなります。

:::message
CCAFの選択肢は「完全に間違っている」ものはほぼなく、「この文脈では次善手」というものが多いです。判断基準となる文脈の読み取りができるかどうかが合否を分けます。
:::

### 3. 判断基準となる英語表現を抽出

試験は英語で実施されます。選択肢の英語ニュアンスを素早く読み解けることが重要で、私は以下のような「フレーズの判断基準」を整理しました。

**ベストプラクティス原則**

| フレーズ                                          | 意味                                     |
| ------------------------------------------------- | ---------------------------------------- |
| addresses the root cause                          | 根本原因に対処する                       |
| make it impossible rather than merely discouraged | 確率的に防ぐのではなく設計で封じる       |
| deterministic guarantee                           | 決定論的な保証（コードや設定で強制）     |
| principle of least privilege                      | 最小権限の原則                           |
| graceful degradation with transparency            | 部分失敗でも価値を提供し、状況を開示する |
| upfront task partitioning                         | 事前のタスク分割（並行性を維持）         |
| separation of concerns                            | 関心の分離                               |

**アンチパターン原則**

| フレーズ                              | 意味                                |
| ------------------------------------- | ----------------------------------- |
| treats only the symptom               | 症状にしか対処していない            |
| probabilistically interpret           | 確率的に解釈する（LLM依存で不安定） |
| masking errors as success             | エラーを成功として偽装する          |
| over-engineers the solution           | 過剰設計                            |
| redistributes which findings get lost | 見落とす情報の場所を変えるだけ      |

これらをチートシートとして整理しておくと、本番で選択肢を見た瞬間に「正解寄り／不正解寄り」の嗅覚が働くようになります。

---

## 試験の傾向と対策

公式の5ドメインとは別に、実際の出題内容を4テーマに整理しました。

### S1: Structured Output / JSONフォーマット

よく出るのが「`tool_use`で取得したJSONの扱い方」です。

**重要な原則：`tool_use`は構造を保証するが、値の正しさは保証しない**

```
tool_use → JSON構造は正しい（フォーマットの問題はない）
         → 値の内容は正しいとは限らない（例：金額が0になっている）
```

Few-Shotプロンプトとpost-tool検証（programmatic check）の使い分けも頻出です：

| 問題の種類                           | 対応策                       |
| ------------------------------------ | ---------------------------- |
| JSONのフォーマットが崩れている       | Few-Shotプロンプトで例示する |
| 値の内容がビジネスルールを満たさない | tool_use後にコードで検証する |

「JSONが返ってきているのに内容がおかしい」ときに、さらにFew-Shotを増やそうとするのはanti-patternです。構造の問題と値の問題は別レイヤーで対処します。

### S2: Claude Code設定・ツール（得点率52%：最難関ドメイン）

自分が最も苦戦したドメインです。知識が細かく、「実在しない機能」が選択肢に混ざってきます。

**6ビルトインツール**

| ツール | 用途                 |
| ------ | -------------------- |
| Read   | ファイルの読み取り   |
| Write  | ファイルの新規作成   |
| Edit   | ファイルの部分編集   |
| Grep   | テキスト検索         |
| Glob   | ファイルパターン検索 |
| Bash   | シェルコマンド実行   |

各ツールの用途を混同した引っかけが出ます。

**CLAUDE.mdのスコープ**

| ファイル                | 適用範囲                       | 典型的な内容                   |
| ----------------------- | ------------------------------ | ------------------------------ |
| `~/.claude/CLAUDE.md` | ユーザーレベル・全プロジェクト | 個人の好み・エディタ設定       |
| `.claude/CLAUDE.md`   | プロジェクト全員               | コーディング規約・チームルール |

プロジェクトレベルに個人設定を書くとチームメンバー全員に強制してしまいます（anti-pattern）。

**設定手段の使い分け**

| 設定手段                  | いつ適用               | ユースケース                           |
| ------------------------- | ---------------------- | -------------------------------------- |
| CLAUDE.md                 | 全会話で常にロード     | コーディング規約・プロジェクト背景     |
| `.claude/rules/` + glob | 該当ファイルを扱うとき | `*.tsx`用規約・`**/*.test.*`用方針 |
| Skills（オンデマンド）    | コマンド呼び出し時のみ | PR review・deploy・migration           |

:::message
「タスクの種類（PRレビューやdeploy）をglobパターンで表現しようとする」のはanti-patternです。ファイルパスパターンに対応しないタスクはSkillsで定義します。
:::

**Skillsのfrontmatter（存在するキー vs 存在しないもの）**

```yaml
# 存在するキー（試験頻出）
name: my-skill
description: "何をするスキルか"
argument-hint: "<target-file>"   # 引数の存在をユーザーに示す
context: fork                    # 出力をメインコンテキストから分離
allowed-tools: [Read, Grep]      # 使用ツールを制限（impossible化）

# 存在しないキー（引っかけ選択肢）
override: true          # 存在しない
user_overrides: true    # 存在しない
priority: high          # 存在しない
```

`allowed-tools`でツールを制限することで「discouraged（やめてと頼む）」から「impossible（設計で不可能にする）」にできます。

**settings.json**

`settings.json`はツール実行権限とHooksを設定します：

- `allowedTools`：事前承認ツールリスト（パーミッション確認をスキップ）
- `permissions.allow` / `permissions.deny`：詳細なツール実行権限
- `hooks`：4種類のフック設定
  - `PreToolUse`：ツール実行前（確認・ブロックに使う）
  - `PostToolUse`：ツール実行後（検証に使う）
  - `Stop`：会話終了時
  - `SubagentStop`：サブエージェント終了時

プロンプトで「このルールに必ず従って」と書くのはdiscouraged（確率的）。HooksのPreToolUseでブロックするのがdeterministic（決定的）な強制方法です。

**存在しない機能（引っかけ選択肢）**

- `.claude/config.yaml` でセクション分け → 存在しない
- `CLAUDE.md` のキャッシュ機構 → 毎回フルロード（キャッシュ問題は存在しない）
- Claude Codeがユーザーの「学習モデル」を会話をまたいで蓄積する → 存在しない

**エージェント（Subagents）設定**

- Explore agentなどビルトインタイプと、カスタム定義エージェントの使い分け
- `isolation: worktree` でメインツリーを汚染しない設計
- CI/CDでの同セッション自己レビューは確証バイアスの温床 → 別セッション・別実行で行う

### S3: Claude API仕様（会話管理）

APIを使ったチャットシステムの設計問題です。

**会話履歴の持ち方**

Claude APIはstatelessのため、会話履歴はクライアント側でmessages配列として管理します。履歴をどう持つか、どう渡すかが問われます。

**コンテキスト肥大への対処：Progressive summarizationの罠**

:::message alert
要約を繰り返す（Progressive summarization）と、固有名詞・注文番号・金額・日付などの詳細が失われていきます。
:::

対処法は「case factsブロック」としてコンテキスト先頭に不変情報を固定すること：

```
[CASE FACTS - DO NOT SUMMARIZE]
顧客名: 田中太郎
注文番号: ORD-2026-0530
金額: ¥128,000
...
[END CASE FACTS]

[会話履歴 - 要約可能]
...
```

**`stop_reason`の重要性**

ループの終了判定はテキスト内容でなく `stop_reason`フィールドで行います：

```python
# ❌ テキスト内容でループ終了を判定する
if "タスク完了" in response.content:
    break

# ✅ stop_reasonで判定する
if response.stop_reason == "end_turn":
    break
```

### S4: マルチエージェント設計（リサーチエージェント）

マルチエージェントシステムの設計問題は最も出題数が多い印象でした。

**Hub-and-Spoke構造が基本原則**

```
                coordinator
               /     |      \
         agentA   agentB   agentC
```

- subagent → subagent の直接通信は禁止
- 必ずcoordinator経由で情報を集約する
- P2P通信やcoordinatorをbypassするルーティングはanti-pattern

**コンテキストの渡し方**

155Kのコンテキストをそのまま各subagentに渡してはいけません。上流で構造化して、必要な情報だけを渡します（Source reduction）：

```
❌ 155Kのコンテキストをまるごとsubagentに渡す
✅ 上流で構造化して50K以下に絞ってから渡す
```

VectorDBや中間要約も「渡した後に処理する」アプローチで、Source reductionの「渡す前に絞る」とは別物です。

**エラーハンドリングの鉄則**

| 失敗の種類           | 例                       | 対処法                            |
| -------------------- | ------------------------ | --------------------------------- |
| transient（一時的）  | timeout・rate limit・503 | subagentレベルでリトライ          |
| persistent（永続的） | ファイル破損・不正形式   | リトライ不要、即coordinatorに返す |

エラーは隠さず、coordinatorが判断できる情報を付けて返します：

```json
{
  "failure_type": "DOCUMENT_CORRUPTED",
  "attempted_query": "invoice_2026_05.pdf",
  "partial_results": null,
  "alternative_approaches": ["try_cached_version", "request_reupload"]
}
```

---

## 試験当日のメモ

**問題文の形式**：3〜4文、60〜80語程度（公式Practice Examと同水準）。TOEIC900あれば読解コストは低いですが、選択肢の英語ニュアンスに集中できる程度の読解力は必要です。

**肢がきわどい**：「正解に見える選択肢が複数ある」問題が多いです。問題の意図（何を問われているか）を読み取れるかどうかで難易度が激変します。

**120分フルに使いました。** 時間配分のコツ：

- 自信がある問題 → さっさと選んで次へ
- 自信がない問題 → 割り切って仮マーク → 最後に戻る

じっくり全問悩んでいると時間が足りなくなります。自信のない問題で粘りすぎないことが重要です。

---

## 試験結果

**757点/1000点（合格720点）、40/60問正解（正答率67%）** で合格。認定証が届きました：

![認定証](/images/ccaf-exam-report/certificate.png)

[Certificate No: qrigmzvazvpd（検証ページ）](https://verify.skilljar.com/c/qrigmzvazvpd)

| ドメイン                                         | 正答率                  |
| ------------------------------------------------ | ----------------------- |
| Agentic Architecture & Orchestration             | 83%                     |
| Tool Design & MCP Integration                    | 83%                     |
| **Claude Code Configuration & Workflows**  | **52%（要注意）** |
| **Prompt Engineering & Structured Output** | **68%（要注意）** |
| Context Management & Reliability                 | 80%                     |

Claude Code設定とPrompt Engineeringが鬼門でした。どちらも「実在しない機能」や「細かいニュアンスの差」を問う問題が多い印象です。受験前にこの2ドメインを重点的に対策することをお勧めします。

---

## 採点補正について

受験後、結果通知までに**約5日かかりました**。受験直後に届くわけではないので、焦らず待ちましょう。

:::message
**取得日 = 合格通知が届いた日** です（受験日ではありません）。デジタルバッジにもこの日付が記載されます。
:::

:::message alert
結果通知とは別に、「採点方法に誤りがあったため補正した」という趣旨のメールが届きました。**採点補正は実際に起こりえます**。仮に正解率が **72%** を超えていなくても、合格するケースがあることを認識しておいてください。
:::

---

## 迷ったときの最終判断基準

選択肢で迷ったとき、以下の問いかけが効きました：

```
1. 根本原因に対処しているか？（symptomへの対応は不正解寄り）
2. "impossible"「そもそもできない設計」にできているか？（"discouraged"「指示で強制」は不正解寄り）
3. 上流で解決しているか？（Source reductionを優先）
4. エラーを隠していないか？（構造化エラーで開示）
5. 通信はcoordinator経由か？（P2P/bypassは不正解）
6. 最小権限・最小スコープになっているか？
```

コード設計・ツール設計・エラー設計のどれを問われても、根底にある軸は同じです：**最小権限、単一責任、必要最小限**。

この軸がぶれなければ、初見の問題でも選択肢をある程度絞れると思います。

---

## おわりに

CCAFはClaude APIやClaude Codeの「なぜそう設計するのか」を体系的に問う、良問揃いの試験でした。合格に向けて頑張っている方の参考になれば幸いです。
