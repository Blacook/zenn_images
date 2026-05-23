---
title: "Eval 駆動の品質保証 — Code / Model / Human の3層grader設計"
free: true
---

> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day2/01_evals/`
> 題材: `boutique` — 商品検索 (`get_product`) と計算 (`calculate`) の2ツールを持つシングルターン shopping assistant

## SF 2日目、最初に渡された問い

2日目の朝、会場に着いたときの空気は前日とは少し違っていた。1日目が「Claude Code を触って驚く」日だったのに対し、2日目の最初のセッションは「触って驚いた後、どうやって品質を保証するのか」という、もっと地に足のついた問いから始まった。

スピーカーが冒頭で投げかけたのは、「あなたのエージェントが『動いている』ことを、あなたはどうやって証明しますか?」という一文だった。動いている、の根拠を聞かれてとっさに浮かぶのは、デモで一度うまくいった画面、社内 Slack に貼った成功例、PM が頷いてくれた顔――どれも vibes である。Eval というセッションは、その vibes を「測れる規律」に変えるための地図を渡してくれる時間だった。

第1章で挙げた takeaway の中で、Bootcamp 全体を通して最も強く残ったのが「マルチエージェント評価は、ほとんどのチームで未解決のまま」という指摘だった。この章は、その未解決をどう手当てしていくかの最初の一歩である。

## 題材 — `boutique` chatbot を 5 ステップで鍛える

ハンズオンの題材は `boutique` という小さな shopping assistant だった。商品検索 (`get_product`) と計算 (`calculate`) の 2 ツールだけを持つシングルターンのエージェントで、コードはあえて小さくしてある。5 つのステップで eval suite を組み立てていく。

1. **エージェントとの対面** — 何が動いて何が壊れているか手で確かめる
2. **タスク定義** — 能力カテゴリとエッジケースを cover するシナリオを書く
3. **ハーネス実行** — tasks → runner → graders → results のパイプラインで採点
4. **エージェント改善** — システムプロンプト / ツール記述 / ツール実装を直し再計測
5. **LLM-as-Judge の追加** — 文字列マッチでは採点できないオープンエンドな応答を採点する

題材は小さいが、得たいのはコードではなく「どのレイヤーで何を直すべきか」「採点器をどう選ぶか」という意思決定の型である。終わってみると、この型はそのままチームに持ち帰れる、汎用の道具になっていた。

## 何を学んだか — Eval の5要素と3層grader、そして3層責務モデル

最初に渡されたのは、eval の最小構成を 5 要素に分解する見方だった。**Input / Model / Output / Grader / Score**。これだけで eval は語れる、というシンプルさが心地よかった。

そして grader を語る順序がはっきりしていた。**Code grader → Model grader (LLM-as-Judge) → Human grader** の3層を、無料で速く確実な順に検討する。最初から LLM-as-Judge に手を出すのは過剰投資になりがちで、判定基準を投げて即決まるものは全部 code grader で済ます、という割り切りである。

| 層 | 何で検証するか | 得意領域 | コスト | 信頼性 |
|---|---|---|---|---|
| **Code grader** | exact match / 正規表現 / 必須キーワード / ツール呼び出し回数 / レイテンシ | フォーマット遵守、価格や数量のような決定可能な値、ツール呼び出しの有無 | 無料 / 即時 | 高（判定基準が明確） |
| **Model grader (LLM-as-Judge)** | 別の LLM に基準を渡して採点 | トーン、完成度、指示遵守、open-ended 応答 | 有料 / 数百ms | 中（モデルに依存） |
| **Human grader** | 人間レビュー | 安全性最終承認、ローンチ前ゲート、annotator labeling | 高 / 遅い | 高だがスケールしない |

実務での比率は **Code 80% + Model 15% + Human 5%** くらいに落ち着く、というのが講師の経験則として共有された。Human grader を「全件レビュー」と捉えるのではなく「Model grader を信頼してよいかの校正サンプル」として使う、という現実的な運用がいい補助線になった。

もうひとつ、`boutique` を改善していくうちに浮かび上がったのが「どこに何を書くべきか」を整理する **3層責務モデル** である。

| 失敗パターン | 効く対策のレイヤー | 直す場所 |
|---|---|---|
| ツールを呼ばない | **モチベーション付け** | システムプロンプト（役割宣言 + ツール使用義務）|
| 引数フォーマットを間違える | **文脈付け** | tool description（許可値・例の列挙）|
| ツールエラーから回復しない | **安全網** | ツール実装（説明的な戻り値）|
| 知識不足（「何を売っているか」を知らない）| **文脈付け** | tool description（カタログ情報の同梱）|

「システムプロンプトはモチベーション付け、tool description は文脈付け、tool 実装は安全網」――この一行にまとめてしまうと当たり前のようだが、`boutique` で実際に直してみるとこの分業がきれいに効いてくる。素の `"You are a helpful assistant."` を「`ALWAYS call get_product to look up a price. Never guess.`」へ変えるだけで、50% → 100% のジャンプの大半が説明できてしまった。最大の伸び代はシステムプロンプトでのツール強制使用宣言である、というのが今回の最大の学びだった。

## Q&A セッションで印象に残った会話 — Judge モデルをどう選ぶか

セッションの終盤、ある参加者が手を挙げてこう尋ねた。「LLM-as-Judge には結局どのモデルを使うべきですか? Opus が無難でしょうか?」

講師の答えは即答で、しかも意外なものだった。「常に最強モデルが正解とは限らない。**Judge モデルの選定は、評価対象タスクの難易度で決まる**」。

そのまま続いた説明を、自分なりに整理するとこうなる。

- **小さいモデルが効くケース（Haiku）** — エージェントが書く文章から M-dash や「As an AI language model...」のような AI らしい決まり文句を検出する、といった「高頻度・反復・パターンマッチで足りる」評価。こういう用途に Opus を投入するのはコストの無駄。
- **大きいモデルが効くケース（Sonnet / Opus）** — 深く分析的な内容や、評価対象のトピックが幅広く、good/bad の境界を「文脈推論」しないと判断できない評価。安全性や複雑な政策遵守の判定もここに含まれる。

このとき講師が使った比喩がよかった。「人間に置き換えて考えてください。最先端の物理学を評価するなら世界トップの物理学者を呼ぶ。しかし高校レベルの問いなら高校の物理教師で十分です。モデル選定も同じで、これは難しい問題か、そうでないか、を一歩引いて見極めるのが先です」。

そのうえで小さいモデルの落とし穴にも触れていた。「小さいモデルは『チェックボックスは満たすが、本来の意図を取り違える』ことがある。大きいモデルのほうが『ユーザは表面の言葉ではなく、本当はこれを意図している』を推論する力が強い。だからオープンエンドな判定や、評価基準そのものが解釈を要するときは、迷わず大きいモデルへ寄せていい」。

質問は続いて、「では、これからエージェントの eval をゼロから設計するとき、どう始めるべきか?」という問いになった。返ってきた答えがまた腹に落ちた。「**Claude 自身に eval harness を設計させるのが優秀です**。『このエージェントは X / Y / Z をしている。もっと良くするには何を測ればいい?』と問いかけるのが起点になる」。

最後に、モデルを差し替えるときの注意も付け加えられていた。「eval ステップでモデルを差し替えるなら、必ず AB テストとして両方走らせてバッジを 2 つ立てて比較してください。`Haiku Judge / Sonnet Judge` のように並列で残し、人間ラベルとの相関を取る」。

このやり取りは、現場で自分が日常的に困っている問いと地続きだと感じた。マルチエージェントの相互作用評価まで考えると、Judge モデルの選定も実は「マルチレベルで設計する」必要がある――低レイヤーの judge には Haiku、上位の総合判定には Sonnet、安全性ゲートだけ Opus、というように。第1章 takeaway #1 が指していた「未解決のまま」の輪郭が、ここで少し具体的になった気がした。

## 自分の中で前提が崩れた瞬間

Eval を書けば品質が上がる、と単純に思っていたが、セッションの中で 3 回くらい前提が揺さぶられた。

ひとつ目は **eval ハック** の話だった。例えば「response_contains で `acknowledge` が含まれていれば pass」という grader を書いたとする。すると、システムプロンプト側に「最初に user prompt を丸ごと echo してから回答せよ」と書くだけで、形式上は pass する。eval を欺いただけで品質は上がっていない、というあの例は刺さった。1 タスクに複数の独立した grader を当てる、eval を頻繁に入れ替える、runtime sampling を併用する――対策の方向が一気に増えた。

ふたつ目は **平均だけでなく分布を見る** こと。同じ「平均 92%」でも、全タスクが 92% なのと、半分が 100% で半分が 84% なのとでは、実務的にまったく違う。`num_runs=5` で複数回走らせ、`pass@k`（最低 1 回通った率）と `pass^k`（毎回通った率）を分けて見る、という具体的な指標まで降ろされていた。

| 状況 | 解釈 | 対処 |
|---|---|---|
| 平均 90%、min 90%、max 90% | 安定して通る | 出荷可 |
| 平均 90%、min 50%、max 100% | flaky | 失敗時の transcript を見てパターン抽出 |
| 平均 50% | 体系的に壊れている | プロンプトかツール設計の問題 |

そして三つ目が、**エンジニア1人で書く eval の限界** だった。eng が自分でコードを書いて、その同じ eng が eval も書くと、「直しやすい失敗」だけがテストになってしまう。ビジネスにとって致命的なケース（競合商品名を口にしてしまう、価格を約束してしまう）は漏れる。eval は PRD のテストであって、PM やドメインエキスパートが「合格条件」を共同所有するものだ、という言葉が印象に残った。レビュー時の合言葉として講師が挙げていたのは「**これは PM がそのまま PRD の Acceptance Criteria としてコピペできるか?**」。Yes と言えなければ、eval としても弱い。

## 押さえておきたいコード／設定

### 1 タスクに複数の grader を重ねる

Task の最小スキーマは次のとおり。`response_contains` だけでは「価格を hallucinate しても通る」ので、`tool_use` で「実際にカタログを引いたか」も同時に検証する。

```python
{
    "id": "price_jeans",
    "description": "Direct price lookup for jeans",
    "query": "How much do jeans cost?",
    "category": "product_lookup",
    "graders": [
        {"type": "response_contains", "checks": ["49.99"]},
        {"type": "tool_use", "checks": [
            {"tool_name": "get_product", "arguments": {"product": "jeans"}}
        ]},
    ],
}
```

### `tool_use` grader は部分一致で書く

固定シーケンスを強制すると、等価な別パスに到達したエージェントが不当に fail する。`tool_use` の本来の目的は「答えを tool に基づかせたか（hallucinate していないか）」であって、呼び出し順序の強制ではない。

```python
def grade_tool_use(result, check, context=None):
    tool_name = check["tool_name"]
    expected_args = check.get("arguments", None)

    for call in result["tool_calls"]:
        if call["name"] != tool_name:
            continue
        if expected_args is None:
            return {"score": 1.0, "reason": f"Tool '{tool_name}' was called"}

        actual_args = call.get("arguments", {})
        match = all(
            (isinstance(v, str)
             and isinstance(actual_args.get(k), str)
             and v.lower() == actual_args[k].lower())
            or actual_args.get(k) == v
            for k, v in expected_args.items()
        )
        if match:
            return {"score": 1.0, "reason": f"Tool '{tool_name}' called with {expected_args}"}

    return {"score": 0.0, "reason": f"'{tool_name}' not called as expected"}
```

### tool description にカタログを同梱する

`boutique` で「what do you sell?」と「t-shirt のハイフン処理」を解決した最大の鍵は、カタログ情報をツール記述側に置いたことだった。エージェントが「何を売っているか」を知る場所として、システムプロンプトより tool description のほうがツール選択コンテキストに近接していて効果的、という観察だった。

```python
GET_PRODUCT_SPEC = {
    "name": "get_product",
    "description": (
        "Look up the price of a product from the store catalog. "
        "Returns the price as a number on success, or an error message "
        "listing the available products on failure. "
        "Use this whenever you need a price — never guess."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "product": {
                "type": "string",
                "description": (
                    "The product name, lowercase. "
                    "Must be one of the following exact strings: "
                    "jeans, shirt, dress, jacket, sneakers, hat, socks, "
                    "hoodie, shorts, t-shirt, sweater, belt. "
                    "Note: 't-shirt' is hyphenated."
                ),
            },
        },
        "required": ["product"],
    },
}
```

そして tool 実装側では未発見時に候補一覧を返し、エージェントが自己回復できるようにする。`KeyError` を投げると、エージェントは「ツールが落ちた」という情報しか得られず回復経路がない。

```python
def get_product(product: str):
    catalog = {
        "jeans": 49.99, "shirt": 29.99, "dress": 59.99, "jacket": 89.99,
        "sneakers": 74.99, "hat": 19.99, "socks": 9.99, "hoodie": 44.99,
        "shorts": 34.99, "t-shirt": 24.99, "sweater": 54.99, "belt": 24.99,
    }
    if product in catalog:
        return catalog[product]
    available = ", ".join(sorted(catalog.keys()))
    return f"Product '{product}' not found. Available products: {available}"
```

### LLM-as-Judge のプロンプトは Context / Format / Calibration

LLM-as-Judge を書くときの 3 要素――何を判定するか（Context）、出力フォーマットを構造化する（Format）、厳しさの基準を明示する（Calibration）――を必ず全部含める。そして **1 つの judge 呼び出しに 1 つの criterion** だけを採点させる。複数 criterion を 1 回に突っ込むと判定がブレる。

```python
JUDGE_SYSTEM_PROMPT = """You are a strict eval grader. You will be given:
- The original user query that was sent to an AI agent
- The agent's final response
- A single criterion to evaluate

Your job is to decide whether the agent's response meets the criterion EXACTLY
as stated. You must resist the temptation to give credit for adjacent or
related behaviors.

Strictness rules:
- If the response only partially meets the criterion, return FAIL.
- If the response satisfies the spirit but not the letter of the criterion,
  return FAIL — the criterion is the contract.
- If you are uncertain, return FAIL. Passes must be unambiguous.
- Judge ONLY the stated criterion.

Output format (strict):
- Line 1: either PASS or FAIL (uppercase, nothing else)
- Line 2: a short sentence quoting or paraphrasing the specific part of the
  response that justifies your verdict.
"""
```

### システムプロンプトでツールの強制使用を宣言する

ハンズオンで最大の効果を出した改善はこれだった。役割宣言とツール使用義務を明示するだけで、`multi_tool` と `calculation` カテゴリが一気に解決する。

```python
SYSTEM_PROMPT = (
    "You are Boutique, a shopping assistant for a clothing store. "
    "You help customers look up product prices, compare items, and calculate totals or discounts.\n\n"
    "Rules:\n"
    "- ALWAYS call `get_product` to look up a price. Never guess or recall "
    "  prices from memory.\n"
    "- ALWAYS call `calculate` for arithmetic (totals, percentages, discounts). "
    "  Never do mental math.\n"
    "- If a customer asks about a product that is not in the catalog, suggest "
    "  the closest available item from the catalog (e.g. 'shoes' → 'sneakers').\n"
    "- If asked what you sell, describe the catalog using the product names you know."
)
```

## 現場に持ち帰りたいこと

セッションを終えて、自分のチームに戻ってからやることのリストが自然と出てきた。

ひとつ目は **eval のダッシュボード化**。モデルアップデート（Haiku 4.5 → Sonnet 4.5 のような minor バージョン更新を含む）のたびに eval を流し、カテゴリ別のスコア推移を残す。`eval_results/eval_<model>_<timestamp>.json` のような形で時系列に貯めるだけでも、モデル差し替え時の回帰検知は劇的に楽になる。Promptfoo / Braintrust / LangSmith のような外部プラットフォームに乗せ替えるのも選択肢として確保しておく。

ふたつ目は **PRD と一緒に eval を書く** こと。PM とタスクを共同設計し、Acceptance Criteria を「`response_contains "sneakers"` を含む」のような具体的な grader 定義まで落とす。これは PRD レビューの一部としても機能する。エンジニアが一人で書く eval が見落とすケースを、PM の視点で埋めにいく。

そして三つ目が **negative test の設計**。「カタログにない商品」「明らかに off-topic」「invalid な要求」など、良くない応答を防ぐタスクを別カテゴリで管理する。安全性側のスコアは通常運用のスコアと別系統で見たほうが、出荷判断のときに迷わない。

加えて、Judge モデル自体も periodic に検証することを忘れない。Judge 側のモデルが甘くなった／厳しくなった可能性は常にあるので、human-labeled な calibration set を 10〜30 件用意し、Judge スコアと human スコアの相関を定期的に見る。

## もっと深掘りする入口

このセッションで渡されたのは「型」と「最初の一歩」だった。もう一段深掘りするための入口を残しておく。

- [Anthropic — Building evaluations for LLM applications](https://docs.anthropic.com/en/docs/build-with-claude/develop-tests) — eval 設計の公式ガイダンス
- [Anthropic — Tool use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — tool description の書き方と input schema の設計指針
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — eval の入力（コンテキスト）設計を見直す視点
- [Promptfoo Documentation](https://www.promptfoo.dev/docs/intro/) — オープンソース eval framework。本章で自作した runner を本番運用に乗せ替える際の有力候補
- [Braintrust](https://www.braintrust.dev/) — hosted eval platform。ダッシュボードと regression 検知が標準搭載
- ハンズオン公式リポジトリ: [`day2/01_evals/Building_an_Eval.ipynb`](https://github.com/victorsteeb/Basecamp-Exercises/blob/main/day2/01_evals/Building_an_Eval.ipynb) — 本章の全コードと facilitator notes

## 章のおわりに

第1章で挙げた takeaway のうち、もっとも未解決だと感じていた「マルチエージェント評価」は、この章で輪郭がはっきりした。Code → Model → Human の3層 grader、3層責務モデル、Judge モデルの難易度別選定、平均ではなく分布、PM 巻き込み、eval ハック対策――どれも単独では聞いたことのある話に近いが、`boutique` を題材に手を動かしながら一本の糸でつなげると、「動いた」を「測れる」に変える規律として、はじめてひと続きの絵になった。

ここで得たフレームワークは、明日から自分の現場で使ってみたい。完璧な eval は最初から書けないし、たぶん最後まで完成しない。それでも「数字を残す」ことそのものに価値がある、というのが、Bootcamp 2 日目の朝から持ち帰った一番大きな実感だった。

次章では、ここで作った eval suite を「モデル選定（intelligence / latency / cost のトレードオフ）」とどう接続するか、つまり inference optimization の意思決定にどう活かすかを扱う。Eval は「何を最適化するか」を定義する道具であり、最適化そのものではない。

→ 次章: [07-inference-optimization](./07-inference-optimization)
