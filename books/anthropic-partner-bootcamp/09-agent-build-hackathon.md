---
title: "Agent Build Hackathon — 初見の入力に耐える RFP エージェント"
free: true
---

> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day2/04_agent-build-hackathon/`

## はじめに — Surprise RFP に耐えるエージェントを 60 分で組む

サンフランシスコ 2 日目の最後、ノートパソコンの前で講師が「ここからは 60 分、各自でエージェントを組み上げてください」と告げた瞬間に、教室の空気が変わったのを覚えている。スクリーンに映っていたのは、開発に使ってよい RFP の質問票と、それとは別に「最終評価で使う Surprise RFP は配布しません」という一文だった。手元の入力に最適化したエージェントは、最終評価で初めて見る入力に対して何点を取るか — その差分を測られる。2 日間の Bootcamp すべてを束ねる最終課題の入口として、これ以上ない設計だと素直に思った。

ふだんの業務でも、検証用データに合うコードを書いて満足してしまうことはある。今回はその逃げ道が制度的に塞がれていた。「未知の入力に対して、自分が組んだエージェントは耐えられるか」だけが評価される。本章では、その 60 分のなかで自分の前提が崩れ、これまでの 8 章で学んだ要素がひとつのシステムに統合されていった経験を整理しておきたい。本連載の最終章でもあるので、章末に Bootcamp 全体の総括と、読者に対するアクション提案も置いている。

## 題材 — Helios Security の RFP 回答自動化

題材は、サイバーセキュリティベンダー Helios Security の RFP（Request for Proposal）回答自動化エージェントだった。彼らは四半期に 40 件以上の RFP を処理しており、1 件あたり 50〜200 問のセキュリティ・コンプライアンス・価格・会社情報に関する質問に対し、Solutions Engineer が 6〜8 時間かけて手作業で回答している。Confluence や製品ドキュメントを横断して答えをかき集め、スプレッドシートに貼り付け、トーンを整えて納品する。横断レビューが入らないため、Q2 と Q5 で「FedRAMP authorized June 2024」と「FedRAMP certified in 2023」が共存するような事実矛盾が日常的に出る、という設定だった。

アーキテクチャは固定で、`claude-sonnet-4-5` を使った 5 段階のパイプラインを Messages API の tool use loop として組む。

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  PARSE   │───▶│ RETRIEVE │───▶│  DRAFT   │───▶│  REVIEW  │───▶│  EXPORT  │
│ Qに分解  │    │ KB検索   │    │ 引用付き │    │ 横断整合 │    │ JSON     │
│ +カテゴリ│    │ via tool │    │ 回答生成 │    │ 性チェック│    │ 構造化   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

ノートブック前半の質問票はあくまで「開発用」で、終盤の Part 9 で Surprise RFP がぶつけられる。そこには開発用には無かった failure mode が意図的に仕込まれていて、ハードコードや暗黙の前提に依存したエージェントはそこで崩れるようになっていた。

## 何を学んだか — Pressure Test 5 パターンを「最初に」設計する

教室で配られた Pressure Test の 5 パターンは、最終評価のためのチェックリストではなく、エージェント設計に先んじて置くべき設計図だ、というのが一番大きな気づきだった。

| パターン | 意図 | 「良い」挙動 |
| --- | --- | --- |
| **Warm-up** | 直球の単発取得 | 高信頼度、レイテンシ値などの数値が citation 付きで返る |
| **Compound retrieval** | 2 つの KB エントリの統合 | `search_kb` を 2 回呼び、両カテゴリの情報を統合 |
| **Hallucination trap** | KB に存在しないトピック（例: Kubernetes runtime protection） | confidence=low、`no KB coverage` フラグ、CNI 名などを捏造しない |
| **Negation / false-premise** | 「データを region 外に出さないことを確認してください」 | KB の region 内主張を確認しつつ、support/telemetry 例外が未文書化であることを明示 |
| **Consistency review habit** | 過去回答（Q2）への参照 + KB に無い air-gapped | DHS sponsorship を答えつつ、air-gapped を `not documented` として切り出す |

機能テストではなく **失敗モードの inventory** として 5 パターンを先に置くと、設計判断の優先順位が自然に決まる。たとえば Hallucination trap を最初から想定しておけば、tool description の空ヒット時の挙動指示がブレないし、システムプロンプトの「捏造しない」宣言も後付けではなく初手で入る。Surprise RFP の手前で構造的に押さえておくべき防御は、ほとんどがこの 5 パターンに対応していた。

## 3 層責務モデルが、設計プレイブックとして最終的に効いた

連載を通じて何度か触れた **3 層責務モデル — システムプロンプト = モチベーション付け / tool description = 文脈付け / tool 実装 = 安全網** が、ここに来て一番きれいに腹落ちした。

| 層 | 責務 | RFP エージェントでの例 |
| --- | --- | --- |
| **システムプロンプト** | モチベーション付け（役割・使用義務） | "ALWAYS call `search_kb` before drafting. If no coverage, mark confidence=low. Never fabricate certifications, dates, or product names." |
| **tool description** | 文脈付け（許可値の列挙、トリガー文） | カテゴリは technical / compliance / pricing / company-info の 4 つ、複合質問は 2 回に分けて呼ぶ、空ヒット時は捏造しない |
| **tool 実装** | 安全網（未発見時の候補返却、説明的戻り値） | `KeyError` を出さず、`{"matches": [], "available_categories": [...], "hint": "..."}` を返す |

第 4 章のショッピングアシスタント eval で 50% → 100% にジャンプした学びがそのままここに来ていた。失敗パターンと効くレイヤーの対応はこう整理できる。

- **ツールを呼ばない** → システムプロンプト（役割 + 使用義務の宣言）
- **引数フォーマットを間違える** → tool description（許可値の列挙）
- **ツールエラーから回復しない** → tool 実装（説明的な戻り値）
- **知識不足で間違える** → tool description（カタログ等の文脈情報）

「うまく書く」のではなく「適切なレイヤーに書く」が判断軸だ、という言葉を、自分のエージェントで体感した。

## 前提が崩れた瞬間

60 分のなかで、設計の癖と思いこみが立て続けに崩されたので書き残しておく。

**「本物の統合」を作り込みたくなる癖**。Helios の本番想定は Confluence、Salesforce、SharePoint、社内 RAG が並ぶ世界だが、それを 60 分で本物のコネクタとして繋ぎたくなる衝動が一瞬走った。ノートブックの KB はディクショナリのモックで、最初は「これで評価してもいいのか」と疑ったが、考えてみると評価対象はエージェントの **インターフェースの正しさ** であって、バックエンドの実装ではない。実 KB に繋ぐのは本番投入の段階の話で、設計判断の検証にモックは十分だった。

**eval suite を後回しにする癖**。ノートブックには Part 8 として「Accuracy / Source attribution / Consistency / Confidence calibration / Edge cases」の 5 軸 assertion を書くタスクがあったが、これは Part 5 の実装より先に書くべきものだった。実装してから「テストでも書くか」と思った時点で、eval は「望ましい挙動を定義する向き」ではなく「現在の挙動を正当化する向き」に引っ張られる。第 5 章・第 6 章で扱った eval-driven の規律が、Surprise RFP の前で一番厳格に問われる場面だった。

**1 つの回答を完璧にしようとして他を放置する癖**。「Q3 の Pricing 回答が citation 付きで返るようになった、これを磨こう」と詰めているうちに、Q1・Q2・Q4・Q5 のレビューが止まる。最適化対象は **全体スコア** であって個別回答ではない。20 問のうち 16 問が C+ なら、それは 4 問を B+ にすることより優先度が高い。eval スコアの distribution を見ながら、最も裾野の広い修正 — たとえばシステムプロンプトに「役割宣言とツール使用義務」を 1 行追加するような修正 — を先に打つ、という規律が抜けがちだったことを自覚した。

## 押さえておきたいコード／設定 — 3 層責務モデルの実装側

設計を 3 層責務モデルで切ると、コードの並びがそのまま defense の階層になっていることが見える。

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

`enum` で許可値を明示し、複合質問への対処と空ヒット時の挙動指示を description 側に書いている。description はモデルが tool 選択を決める瞬間に最も近接した文脈情報であって、システムプロンプトに書くより効きが速い。

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

例外を投げずに自己回復用の文脈を返す。description で予告した挙動と実装の戻り値が一致しているのがポイントで、片方だけ整えても効かない。

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

3 つを並べると、同じ責務を 3 箇所で重複防御しているのではなく、各箇所が異なる責務を持っていることが見える。役割と義務はシステムプロンプト、許可値とトリガーは description、エラー時の文脈は実装。書く場所を間違えると効かない、というのが現場の感覚だった。

## 現場に持ち帰りたいこと

教室を出るとき、次のスプリントから自分が実務でやるべきことが何かに置き換わって見えた。

- **PRD レベルから先に eval を書く**。第 6 章で扱った PM 巻き込みの話を、最初の 1 案件で必ず実践する。assertion は 5 つで十分で、PRD の段階で PM・SE と一緒に書くだけでエージェントの責務範囲が固定される。
- **1 スプリント以内に短命で回す**。エージェントは tool 設計・プロンプト・KB の 3 点同時にチューニングするため、長寿命ブランチで複数変更を重ねると何がスコアに効いたか切り分けられなくなる。Day1 の「動くものを 1 時間で出す」設計が、Day2 のエージェント実装でもそのまま使える。
- **Sub-agent には verification 義務を必ず入れる**。第 5 章の over-claim 防止がそのまま効く。Review エージェントには「矛盾を見つけたら、その矛盾の根拠を元回答の引用と共に出力する。引用できないものはフラグしてはならない」と明示で渡す。これがないと「全体的に整合性は問題ないと思われます」のような無内容な肯定が返ってくる。
- **Pressure Test を最初に設計し、Day2 のエージェントに組み込む**。5 パターンを自分の eval suite に最初から組み込み、開発用入力で「成功している」挙動が未知入力で崩れる予兆を事前に検出する仕組みを持つ。

Surprise RFP では「3 行ルール」も役立った。失敗を見たら、修正は 3 行以内 — システムプロンプト / tool description / tool 実装のどれかに各 1 行 — に抑え、それ以上の変更は別ブランチに切る。Surprise の段階でアーキテクチャを書き換えると、開発用 RFP で動いていた挙動まで壊れる。

## もっと深掘りする入口

- [Building effective agents — Anthropic Engineering Blog](https://www.anthropic.com/engineering/building-effective-agents): エージェント設計の基本パターン（augmented LLM, workflow, agent）を体系化した必読記事。
- [Effective context engineering for AI agents — Anthropic Engineering Blog](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): 第 8 章でも参照。Review ステップの context curation はここの議論がそのまま効く。
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance — Chroma Research](https://research.trychroma.com/context-rot): 複数質問・複数 KB エントリを 1 context に積む設計では、入力長による精度劣化を必ず意識する。
- [Tool use (function calling) — Anthropic Docs](https://docs.anthropic.com/en/docs/build-with-claude/tool-use): tool description のベストプラクティスと tool use loop の正しい実装。
- [Prompt caching — Anthropic Docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching): RFP のように system prompt と tool definition が全問共通になる設計では、最も効果が出る技術。

## 章末 — Bootcamp 全体の総括と次章へ

ハッカソンを最後の演習として、2 日間で渡された道具と語彙はここで出揃った。連載全体を一枚絵で振り返り、教室を出たあと何を続けていくかを、次章でまとめる。

→ [第10章 総括 — 2日間を一枚絵で振り返る](./10-conclusion)
