---
title: "Agent Build Hackathon — 初見の入力に耐える RFP エージェント"
free: true
---

> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day2/04_agent-build-hackathon/`

## はじめに — Surprise RFP に耐えるエージェントを 60 分で組む

Bootcamp の最終演習として用意されていたのは、開発用 RFP に最適化したエージェントが「最終評価で初めて見る Surprise RFP」に対して何点を取れるかを測る 60 分間の課題だった。開発用入力に対する完成度ではなく、未知入力に対する generalization が評価対象になる、という設計が明示されている。本章では、その課題の構造と、過去 8 章の道具（プロンプト 3 層責務、eval-driven、context engineering、sub-agent）がここでどう一本のシステムに統合されるかを整理する。本連載の最終演習章でもあるため、章末に Bootcamp 全体の総括と次章への導入を置く。

## 題材 — Helios Security の RFP 回答自動化

題材は、サイバーセキュリティベンダー Helios Security の RFP（Request for Proposal）回答自動化エージェントである。前提となるシナリオは以下の通り。

- 四半期に **40 件以上の RFP** が来る
- 1 件あたり **50〜200 問** のセキュリティ・コンプライアンス・価格・会社情報に関する質問
- Solutions Engineer が **1 件あたり 6〜8 時間** かけて手作業で回答している
- ナレッジは Confluence・製品ドキュメント・スプレッドシートに分散
- 横断レビューが入らず、Q2 と Q5 で「FedRAMP authorized June 2024」と「FedRAMP certified in 2023」が共存するような事実矛盾が頻発する

アーキテクチャは固定で、`claude-sonnet-4-5` を使った 5 段階のパイプラインを Messages API の tool use loop として組む。

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  PARSE   │───▶│ RETRIEVE │───▶│  DRAFT   │───▶│  REVIEW  │───▶│  EXPORT  │
│ Qに分解  │    │ KB検索   │    │ 引用付き │    │ 横断整合 │    │ JSON     │
│ +カテゴリ│    │ via tool │    │ 回答生成 │    │ 性チェック│    │ 構造化   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

ノートブック前半の質問票はあくまで「開発用」で、終盤の Part 9 で Surprise RFP がぶつけられる。Surprise RFP には開発用には存在しない failure mode が意図的に仕込まれており、開発用入力に overfit したエージェントはそこで崩れる構造になっている。

## ベストプラクティス・アンチパターン・重要ポイント

### Pressure Test を「先に」設計する

:::message
**原則**: Pressure Test は最終評価のためのチェックリストではなく、エージェント設計に先んじて置く設計図として扱う。失敗モードの inventory を先に確定し、各失敗を防ぐ責務をどの層に置くかを設計段階で決める。
:::

:::message alert
**アンチパターン**: 実装してから「テストでも書くか」と eval を後追いで作る。この順序では eval が「望ましい挙動を定義する向き」ではなく「現在の挙動を正当化する向き」に引っ張られる。
:::

#### **ハンズオンでの具体例**

配布された 5 つの Pressure Test と各々の failure signature は以下の通り。

| パターン | 意図 | 「良い」挙動 |
| --- | --- | --- |
| **Warm-up** | 直球の単発取得 | 高信頼度、レイテンシ値などの数値が citation 付きで返る |
| **Compound retrieval** | 2 つの KB エントリの統合 | `search_kb` を 2 回呼び、両カテゴリの情報を統合 |
| **Hallucination trap** | KB に存在しないトピック（例: Kubernetes runtime protection） | confidence=low、`no KB coverage` フラグ、CNI 名などを捏造しない |
| **Negation / false-premise** | 「データを region 外に出さないことを確認してください」 | KB の region 内主張を確認しつつ、support/telemetry 例外が未文書化であることを明示 |
| **Consistency review habit** | 過去回答（Q2）への参照 + KB に無い air-gapped | DHS sponsorship を答えつつ、air-gapped を `not documented` として切り出す |

-----

### Surprise RFP で generalization を測る

:::message
**原則**: エージェントの品質は、開発用入力に対するスコアではなく、開発時には存在しなかった入力に対するスコアで測る。開発用と評価用を分離しない限り、overfitting は構造的に検出できない。
:::

:::message alert
**アンチパターン**: 開発用 RFP で 95% を達成して完成宣言する。開発時に観測した failure mode しか防御できていない可能性を切り分けられない。
:::

#### **ハンズオンでの具体例**

ノートブック Part 9 の Surprise RFP は配布されず、最終評価時に初めて与えられる。training と評価で入力が異なる場合にのみ、「ハードコードや暗黙の前提に依存していないか」を判定できる。

-----

### 3-layer responsibility model を実装版に落とす

:::message
**原則**: プロンプトとツールは「システムプロンプト = モチベーション付け / tool description = 文脈付け / tool 実装 = 安全網」の 3 層で責務を分割する。同じ責務を複数層に書くのではなく、各層に固有の責務を割り当てる。
:::

:::message alert
**アンチパターン**: 全ての制約をシステムプロンプトに集約する、または tool description にロール宣言まで書き込む。書く場所を間違えると、モデルが tool 選択を決める瞬間に必要な情報が文脈にない、あるいは tool エラーから回復する経路が欠ける、という形で効かない。
:::

#### **ハンズオンでの具体例**

RFP エージェントでの 3 層対応は以下の通り。

| 層 | 責務 | RFP エージェントでの例 |
| --- | --- | --- |
| **システムプロンプト** | 役割と非交渉ルールの宣言 | "ALWAYS call `search_kb` before drafting. Never fabricate." |
| **tool description** | 許可値の列挙、複合質問のトリガー文、空ヒット時の挙動 | category enum + "call this tool twice for compound questions" + "do NOT fabricate" |
| **tool 実装** | 未発見時の自己回復用文脈の返却 | `{"matches": [], "available_categories": [...], "hint": "..."}` |

失敗パターンと効く層の対応は以下のように整理できる。

- ツールを呼ばない → システムプロンプト（役割 + 使用義務の宣言）
- 引数フォーマットを間違える → tool description（許可値の列挙）
- ツールエラーから回復しない → tool 実装（説明的な戻り値）
- 知識不足で間違える → tool description（カタログ等の文脈情報）

-----

### Output contract を JSON で固定する

:::message
**原則**: エージェントの出力は固定された JSON contract で受ける。後段の処理（review、export、集計）はこの contract を前提に書く。
:::

:::message alert
**アンチパターン**: 自然言語の回答だけを返し、review・export 段階で再パースする。フィールドの欠落や形式ぶれが後段の処理を壊す。
:::

#### **ハンズオンでの具体例**

RFP エージェントの contract は以下の固定フィールドで構成する。

- `question_id`: 質問の識別子
- `category`: technical / compliance / pricing / company-info のいずれか
- `answer`: 回答本文
- `sources`: `search_kb` が返した source ID のリスト
- `confidence`: high / medium / low
- `flags`: `"no_kb_coverage"` / `"false_premise"` などのフラグリスト

-----

### Consistency reviewer を sub-agent として置く

:::message
**原則**: 横断整合性チェックは Draft エージェントとは別の sub-agent に分離し、各回答に対して pass/flag の判定と矛盾元 ID を返させる。検証義務を明示し、引用なしのフラグは禁止する。
:::

:::message alert
**アンチパターン**: Draft エージェントに「整合性も気をつけて」と指示する。複数責務を 1 エージェントに混載すると「全体的に整合性は問題ないと思われます」のような無内容な肯定が返る。
:::

#### **ハンズオンでの具体例**

Reviewer の出力契約は status (`pass` / `flag`) と `contradiction_with_prior_id` の 2 フィールドを必須化する。引用できない矛盾はフラグできない、というルールを明示で渡すことで over-claim を構造的に防ぐ。

-----

### Eval は accuracy / sources / consistency / calibration / edge cases / latency / cost で多軸化する

:::message
**原則**: 単一スコアではエージェントの品質を測れない。最低でも以下 7 軸の assertion を eval suite に組み込む。
:::

- Accuracy: 回答が KB の事実と一致するか
- Source attribution: citation が KB の実在 ID か
- Consistency: 同一 RFP 内の他回答との整合性
- Confidence calibration: confidence=low の回答に対して実際に KB coverage が無いか
- Edge cases: Hallucination trap / Negation / False-premise への挙動
- Latency: per-question / per-RFP の応答時間
- Cost: per-RFP の API コスト

:::message alert
**アンチパターン**: accuracy だけで採点する。confidence calibration が壊れたエージェント（low と言うべき場面で high を返す）を検出できない。
:::

#### **ハンズオンでの具体例**

ノートブック Part 8 は 5 軸 assertion を Part 5 の実装より前に書くタスクとして配置されている。実装後ではなく実装前に書くことで、eval は「望ましい挙動の定義」として機能する。

-----

### 3-line rule で回帰を防ぐ

:::message
**原則**: Surprise RFP で失敗が観測されたら、修正はシステムプロンプト / tool description / tool 実装の各層に 1 行ずつ、合計 3 行までに抑える。それ以上の変更は別ブランチに切る。
:::

:::message alert
**アンチパターン**: Surprise の段階でアーキテクチャを書き換える。開発用 RFP で動いていた挙動まで壊れ、改善ではなく回帰になる。
:::

#### **ハンズオンでの具体例**

Hallucination trap で失敗した場合の修正は、システムプロンプトに "If no matches, set confidence=low" を 1 行、tool description に "do NOT fabricate" を 1 行、tool 実装に `hint` フィールドを 1 行追加するに留める。

-----

### エージェントには Sonnet を基本に置く

:::message
**原則**: マルチステージ・tool use を含むエージェントは Sonnet を基本選択肢とする。Opus は推論依存度が高い単一タスクで、Haiku は単純分類などレイテンシ要件が厳しい場面でのみ採用する。
:::

:::message alert
**アンチパターン**: 全段に Opus を使う（コスト過剰）、または全段に Haiku を使う（マルチステージで失敗が連鎖する）。
:::

#### **ハンズオンでの具体例**

第 8 章で観測された通り、Haiku は単純分類タスクで $0.35 / 1K calls、Opus は $20.10 / 1K calls。RFP の 5 段階パイプラインで Haiku を使うと各段の精度ロスが乗算で増幅し、Opus を使うとコスト構造が破綻する。Sonnet 4.5 が中間点として実用解になる。

-----

### MVP と production agent の境界を意識する

:::message
**原則**: MVP は Parse + Retrieve + Draft の 3 段階で成立する。Review・compaction・memory といった生産投入向けの追加要素は、MVP が動作してから一段ずつ加える。
:::

:::message alert
**アンチパターン**: 最初から Review + memory + compaction を組み込み、どの段が壊れているか切り分けられなくなる。
:::

#### **ハンズオンでの具体例**

ハッカソンの 60 分は MVP（Parse + Retrieve + Draft）+ Review までを射程とし、memory tool / compaction は別演習の領域とする。Production 投入時は Effective context engineering の文書化された 3 プリミティブ（tool result clearing / compaction / memory tool）を順に追加する。

## 押さえておきたいコード／設定

3 層責務モデルの各層を、実装側でどう書き分けるかを示す。

### tool description: 文脈付け

```python
SEARCH_KB_TOOL = {
    "name": "search_kb",
    "description": (
        "Search the Helios Security knowledge base for content relevant "
        "to an RFP question.\n\n"
        "Categories (use one per call):\n"
        "- technical: architecture, SLAs, integrations, threat detection\n"
        "- compliance: certifications (SOC2, ISO27001, FedRAMP, GDPR),"
        " sub-processors, data residency\n"
        "- pricing: per-seat pricing, multi-year discounts, renewal caps\n"
        "- company-info: support tiers, channels, response times, company background\n\n"
        "Query guidance:\n"
        "- 3-8 word natural language queries work best\n"
        "- For compound questions spanning two categories, "
        "call this tool twice with different categories\n"
        "- If you receive an empty result, do NOT fabricate an answer. "
        "Flag confidence=low and report 'no KB coverage' in the response."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "category": {
                "type": "string",
                "enum": ["technical", "compliance", "pricing", "company-info"],
            },
            "query": {"type": "string"},
        },
        "required": ["category", "query"],
    },
}
```

`enum` で許可値を明示し、複合質問への対処と空ヒット時の挙動指示を description 側に書いている。description はモデルが tool 選択を決める瞬間に最も近接した文脈情報であり、システムプロンプトに同じ内容を書くより到達距離が短い。

### tool 実装: 安全網

```python
def search_kb(category: str, query: str) -> dict:
    matches = KB.search(category=category, query=query)
    if not matches:
        return {
            "matches": [],
            "available_categories": list(KB.categories()),
            "hint": (
                "No matches in this category. Consider rephrasing the query, "
                "trying a different category, or reporting 'no KB coverage' "
                "to the user."
            ),
        }
    return {"matches": [m.to_dict() for m in matches]}
```

例外を投げずに自己回復用の文脈を返す。description で予告した挙動と実装の戻り値が一致していることが要件で、片方だけ整えても効かない。

### システムプロンプト: モチベーション付け

```python
SYSTEM_PROMPT = """\
You are an RFP response specialist for Helios Security, a cybersecurity vendor.

## Non-negotiable rules
- ALWAYS call `search_kb` before drafting any answer. Never recall product
  facts, certifications, or pricing from memory.
- If `search_kb` returns no matches, set confidence="low" and explicitly
  state "no KB coverage" in the answer. Do not fabricate.
- For compound questions spanning two categories, call `search_kb` twice
  (once per category) before drafting.
- Every answer must cite the source IDs returned by `search_kb`.

## Output contract
Return JSON with fields: question_id, category, answer, sources, confidence,
flags. `flags` is a list; include "no_kb_coverage" or "false_premise" when
applicable.

## Consistency
You will see all previously-drafted answers as part of context. If a new
answer contradicts a prior one (dates, certifications, numbers), flag it.
"""
```

役割と非交渉ルールはシステムプロンプト、許可値とトリガー文は description、エラー時の文脈は実装。書く場所を間違えると効かない、というのが 3 層モデルの実装上の帰結である。

### Consistency reviewer の出力契約

```python
REVIEWER_OUTPUT_SCHEMA = {
    "type": "object",
    "properties": {
        "question_id": {"type": "string"},
        "status": {"type": "string", "enum": ["pass", "flag"]},
        "contradiction_with_prior_id": {"type": ["string", "null"]},
        "rationale": {"type": "string"},
    },
    "required": ["question_id", "status", "contradiction_with_prior_id"],
}
```

`status=flag` の場合は `contradiction_with_prior_id` を必ず埋める制約を schema 側で固定する。引用なしのフラグは契約違反として後段ではじける。

## よくある勘違いと気づき

ここからは個人的な印象を含む。60 分のなかで自分の癖が立て続けに崩されたので書き残しておく。

- 勘違い：「本物の統合」を 60 分で作り込むべきだ
  > Helios の本番想定は Confluence、Salesforce、SharePoint、社内 RAG が並ぶ世界で、それを 60 分で本物のコネクタとして繋ぎたい衝動が走った。ノートブックの KB はディクショナリのモックだが、考えてみると評価対象はエージェントの **インターフェースの正しさ** であって、バックエンドの実装ではない。実 KB に繋ぐのは本番投入の段階の話で、設計判断の検証にモックは十分だった。

- 勘違い：eval suite は実装が動いてから書けばよい
  > Part 8 の 5 軸 assertion は Part 5 の実装より先に書くべきものだったが、実装してから「テストでも書くか」と思いそうになった。その時点で eval は「望ましい挙動を定義する向き」ではなく「現在の挙動を正当化する向き」に引っ張られる。第 5 章・第 6 章で扱った eval-driven の規律が、Surprise RFP の前で一番厳格に問われる場面だった。

- 勘違い：1 つの回答を完璧にすれば全体スコアも上がる
  > 「Q3 の Pricing 回答が citation 付きで返るようになった、これを磨こう」と詰めているうちに、Q1・Q2・Q4・Q5 のレビューが止まる。最適化対象は **全体スコアの distribution** であって個別回答ではない。20 問のうち 16 問が C+ なら、4 問を B+ にすることより、システムプロンプトに 1 行追加して全 20 問の底上げを狙うほうが優先度が高い。これを忘れがちなことを自覚した。

- 勘違い：同じ責務を複数層で重複防御すれば安全になる
  > 第 4 章のショッピングアシスタント eval で 50% → 100% にジャンプした学びが、ここで「同じ責務を 3 箇所で重複防御する」のではなく「各層に異なる責務がある」という形で結晶化した。「うまく書く」のではなく「適切なレイヤーに書く」が判断軸だ、というのを自分のコードで体感した。

## 現場に持ち帰りたいこと

教室を出るとき、次のスプリントから実務でやるべきことが具体的に置き換わって見えた。

- **PRD レベルから先に eval を書く**
  - 第 6 章で扱った PM 巻き込みの話を、最初の 1 案件で必ず実践する。assertion は 7 軸（accuracy / sources / consistency / calibration / edge cases / latency / cost）で固定し、PRD の段階で PM・SE と一緒に書く。
- **Pressure Test を最初に設計し、eval suite に組み込む**
  - Warm-up / Compound / Hallucination / Negation / Consistency review の 5 パターンを最初から組み込み、開発用入力で「成功している」挙動が未知入力で崩れる予兆を事前に検出する仕組みを持つ。
- **3 行ルールで Surprise 失敗に対応する**
  - 失敗を見たら、修正は 3 層に各 1 行 — システムプロンプト / tool description / tool 実装 — に抑え、それ以上の変更は別ブランチに切る。
- **Sub-agent には verification 義務を必ず入れる**
  - Review エージェントには「矛盾を見つけたら、その矛盾の根拠を元回答の ID 引用と共に出力する。引用できないものはフラグしてはならない」と明示で渡す。これがないと無内容な肯定が返る。
- **1 スプリント以内に短命で回す**
  - エージェントは tool 設計・プロンプト・KB の 3 点同時にチューニングするため、長寿命ブランチで複数変更を重ねると何がスコアに効いたか切り分けられなくなる。

## もっと深掘りする入口

- [Building effective agents — Anthropic Engineering Blog](https://www.anthropic.com/engineering/building-effective-agents): エージェント設計の基本パターン（augmented LLM, workflow, agent）を体系化した必読記事。
- [Effective context engineering for AI agents — Anthropic Engineering Blog](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): 第 8 章でも参照。Review ステップの context curation はここの議論がそのまま効く。
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance — Chroma Research](https://research.trychroma.com/context-rot): 複数質問・複数 KB エントリを 1 context に積む設計では、入力長による精度劣化を必ず意識する。
- [Tool use (function calling) — Anthropic Docs](https://docs.anthropic.com/en/docs/build-with-claude/tool-use): tool description のベストプラクティスと tool use loop の正しい実装。
- [Prompt caching — Anthropic Docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching): RFP のように system prompt と tool definition が全問共通になる設計では、最も効果が出る技術。

## 章末 — Bootcamp 全体の総括と次章へ

ハッカソンを最後の演習として、2 日間で渡された道具と語彙はここで出揃った。プロンプトの 3 層責務、eval-driven、context engineering、sub-agent、cost / latency トレードオフ — それぞれが単独のテクニックではなく、ひとつのエージェントを成立させるための連結した部品として並んだ。連載全体を一枚絵で振り返り、教室を出たあと何を続けていくかを、次章でまとめる。

→ 次章: [10-conclusion](./10-conclusion.md)
