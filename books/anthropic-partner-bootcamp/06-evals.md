---
title: "Day2-01: Eval 駆動の品質保証 — 5要素パイプラインと3層Grader、3層責務モデル"
free: true
---
> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day2/01_Evals/Building_an_Eval.ipynb`
> 題材: `boutique` — `get_product` と `calculate` の2ツールを持つシングルターン shopping assistant

## はじめに — 「動いた」を「測れる」に変える

Bootcamp 2 日目の開幕セッションは、Day 1 の「Claude Code を触って驚く」モードから一気にシフトして、「そのエージェントの挙動が品質的に正しいことをどう証明するか」という問いから始まった。デモでの好評、PM の頷いた顔――そういったなんとなく良さそうという「vibes（雰囲気）」を、データで語れる規律に置き換えるための地図を渡されるのがこの章のテーマである。本章では Eval（エージェントの応答を自動採点する評価スクリプト）を 5 要素パイプラインに分解し、3 層 Grader（採点器）と 3 層責務モデルという 2 つの軸で boutique chatbot を鍛える流れを追っていく。

## 題材 — `boutique` chatbot を 5 ステップで鍛える

題材は `boutique` というシングルターンの shopping assistant で、あえてシンプルに作られたもの。tools は 2 つだけで、`get_product(product)` が商品名から価格を引き、`calculate(op, input1, input2)` が四則演算を担う。ループは `User query → Claude decides → Tool call → Tool result → ... → Final answer` のシンプルな単一ターン構成。

ハンズオンはこの題材を題目に、Eval suite（評価スクリプトをまとめたテストセット）を 5 ステップで組み立てる構成になっている。

1. **エージェントとの対面** — 現状の挙動と壊れている箇所を確かめる
2. **タスク定義** — 能力カテゴリとエッジケースを cover するシナリオを書く
3. **ハーネス実行** — Eval を自動実行するフレームワーク（ハーネス）で `Tasks（テストケース） → Runner（実行エンジン） → Graders（採点器群） → Results` のパイプラインを流す
4. **エージェント改善** — system prompt / tool description / tool 実装を直し再計測
5. **LLM-as-Judge の追加** — 別の LLM を審判役に使い、文字列マッチでは採点できないオープンエンドな応答を採点する

題材はシンプルだが、得たいのはコードではなく「どのレイヤーで何を直すべきか」「採点器をどう選ぶか」という意思決定の型である。Day 2 開幕講義として、その型を一気通貫で渡すための題材として構成されている。

## ベストプラクティス・アンチパターン・重要ポイント

### Eval は Input/Model/Output/Grader/Score の5要素に分解する

:::message
**原則**: Eval の最小構成は `Input / Model / Output / Grader / Score` の 5 要素である。最初の 3 つはこれまでの推論パイプラインそのものであり、Eval が追加するのは「Output を Grader に渡して Score を返す」最後の 2 ステップだけである。この最小単位を維持することで、評価対象がモデルなのか、Grader なのか、タスク設計なのかを切り分けやすくなる。
:::

:::message alert
**アンチパターン**: 「何を評価しているか」を明確にしないまま Eval スクリプトを書き始めると、Grader の責務とタスク設計が混ざり、結果の解釈が不能になる。Runner に表示処理を埋め込んだり、Grader が複数の独立した観点を 1 関数で採点したりすると、再利用も差し替えも難しくなる。
:::

#### **ハンズオンでの具体例**

公式ハーネスは Runner と Grader を明示的に分離している。`run_Eval()` はデータ構造を返すだけで、`print_summary()` は別関数。Grader は `Grader_REGISTRY` という dict にプラグインされる構造で、新しい Grader タイプは「1 関数 + 1 dict エントリ」で追加できる。agent も `agent_fn` 引数で受け取り、mock 差し替えや instrumentation を可能にする。

> 公式ハーネスの実装コンポーネント（`Tasks → Runner → Graders → Results`）は、この 5 要素を実装レベルで対応させたものである。

| 5 要素（概念） | 実装コンポーネント                       |
| -------------- | ---------------------------------------- |
| Input          | Tasks の `query` フィールド              |
| Model          | Runner が呼ぶ `agent_fn`                 |
| Output         | Runner が収集する transcript             |
| Grader         | Graders（`Grader_REGISTRY` の関数群）    |
| Score          | Results の `grade` / `passed` フィールド |

> なお Tasks は Input（クエリ）だけでなく「どの Grader をどう適用するか」の定義も内包する。また実装上は Runner（`run_single_task()`）が Model 実行・Output 収集・Grader 適用をすべて担うため、図の `Runner → Graders` は「別プロセス」ではなく Runner 内部の処理ステップと捉えるのが正確である。

---

### 3 層 Grader を組み合わせる

:::message
**原則**: Grader は **Code → Model → Human** の順に検討する。判定基準が明確で決定的に書けるものは全部 code Grader で済ませ、open-ended なものだけ LLM-as-Judge に持ち上げ、最終ゲートだけ human にする。実務上の比率の目安は **Code 80% / Model 15% / Human 5%**。
:::

| 層                              | 何で検証するか                                                 | 得意領域                                                          | コスト        | 信頼性               |
| ------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------- | ------------- | -------------------- |
| **Code Grader**                 | exact match / regex / 必須キーワード / tool 呼び出し / latency | フォーマット遵守、価格や数量など決定可能な値、tool 呼び出しの有無 | 無料 / 即時   | 高（基準が明確）     |
| **Model Grader (LLM-as-Judge)** | 別 LLM に基準を渡して採点                                      | トーン、完成度、指示遵守、open-ended 応答                         | 有料 / 数百ms | 中（モデル依存）     |
| **Human Grader**                | 人間レビュー                                                   | 安全性最終承認、ローンチ前ゲート、Judge 校正サンプル              | 高 / 遅い     | 高だがスケールしない |

:::message alert
**アンチパターン**: 最初から LLM-as-Judge に全タスクを通すのは過剰投資である。exact match や `response_contains` で決着するタスクに Judge 呼び出しを乗せると、コストが線形に増え、Grader 自身の非決定性で Eval が不安定（flaky）になる。逆に Human Grader を「全件レビュー」と捉えるのも誤りで、Human はスケールしない。
:::

#### **ハンズオンでの具体例**

公式ハーネスの code Grader は `response_contains`（最終応答に文字列を含むか）、`response_numeric`（数値が許容範囲内か）、`tool_use`（特定の tool を特定の引数で呼んだか）の 3 種。各 Grader はバイナリスコア（0/1）と理由を返し、タスクは「全 Grader / 全 check の AND」で pass する。Human Grader は出荷ゲートと、Model Grader を信頼してよいかの校正サンプル（10〜30 件）として運用する。

---

### Judge モデルは「審判の専門性」で選ぶ

:::message
**原則**: LLM-as-Judge のモデル選定は「常に最強モデル」が正解ではない。**評価対象タスクの難易度**で決める。高頻度・反復・パターンマッチで足りる評価には Haiku、文脈推論や open-ended な解釈を要する評価には Sonnet / Opus を充てる。
:::

:::message alert
**アンチパターン**:

- 「念のため Opus にしておく」と全 Judge を最上位モデルにすると、Judge コストが本番推論より高くなる。
- 逆に難しい評価まで Haiku で済ませると、「チェックボックスは満たすが本来の意図を取り違える」誤判定が増える。
- Judge モデルを差し替える際に AB テストを取らないと、スコア変動が「エージェントの劣化」なのか「Judge の厳しさ変化」なのか切り分けられない。
  :::

#### **ハンズオンでの具体例**

- **Haiku が向くケース**: 応答中の M-dash や `As an AI language model...` のような AI 定型句の検出など、パターンマッチで判定可能な評価。
- **Sonnet / Opus が向くケース**: 安全性、複雑な政策遵守、open-ended な「適切な応答か」判定など、文脈推論を要する評価。
- アナロジー: 「最先端物理学の評価には世界トップの物理学者を、高校レベルの問いには高校の物理教師を呼ぶ」――Judge も同じく、難易度で人選を変える。
- モデル差し替え時は `Haiku Judge / Sonnet Judge` のように AB テストで両方走らせ、human-labeled な校正セットとの相関を取る。

---

### 3層責務モデル: 失敗パターン（failure）を修正箇所（fix location）にマップする

:::message
**原則**: エージェントの失敗は「どこを直すか」で **モチベーション付け / 文脈付け / 安全網** の 3 層に振り分けられる。失敗パターンをこの層にマップすることで、修正先が決定される。
:::

| 失敗パターン               | 有効な対策層       | 修正箇所                                   |
| -------------------------- | ------------------ | ------------------------------------------ |
| ツールを呼ばない           | モチベーション付け | system prompt（役割宣言 + ツール使用義務） |
| 引数フォーマットを間違える | 文脈付け           | tool description（許可値・例の列挙）       |
| ツールエラーから回復しない | 安全網             | tool 実装（説明的な戻り値）                |
| 知識不足                   | 文脈付け           | tool description（カタログ等の文脈情報）   |

:::message alert
**アンチパターン**: system prompt にカタログを列挙する、tool description に役割宣言を書く、tool 実装で `KeyError` を投げるだけで終わる――いずれも層を取り違えた修正であり、エージェントが自己回復できない経路を残す。`empty list` や生の例外をそのまま返すと、エージェントは「ツールが落ちた」以上の情報を得られない。
:::

#### **ハンズオンでの具体例**

`boutique` の `get_product` で未発見時に `KeyError` を投げる代わりに `{"available_products": [...], "hint": "..."}` 相当の「候補一覧を含むエラーメッセージ文字列」を返す設計にすると、エージェントは類義語（synonym: `shoes` → `sneakers`）への自己回復ルートを得る。system prompt 側は `"You are a helpful assistant."` から `"ALWAYS call get_product to look up a price. Never guess."` に書き換える。boutique では 50% → 100% への伸びの大半がこの system prompt のツール強制使用宣言で説明される。

---

### 構築と評価は別人格で書く

:::message
**原則**: build を書いた人格が同じく Eval も書くと、無意識のうちに「自分が作ったものが通る方向」に Grader が寄る。Eval は build と切り離した責務として、最初から別の場所・別の人・別の時間で書く。
:::

:::message alert
**アンチパターン**: 同一エンジニアが build と Eval を同タイミングで書くと、「直しやすい失敗」だけがテスト化され、ビジネスに致命的なケース（競合商品名を口にする、価格を約束する、off-topic）が漏れる。Grader が build の言い訳を内包してしまう状態である。
:::

#### **ハンズオンでの具体例**

- **コードの場所を分ける**: 同一リポジトリでも `agent/` と `Evals/` を並列に立てる。
- **書く人と時間を分ける**: 同一チーム内でも build と Eval を別人が書く（人間でもAIでも）。同一人物しかいない場合は「build を書いた翌日に Eval を書く」だけでも認知バイアスが下がる。
- **責務オーナーを分ける**: PM / SE を Eval 工程に巻き込む。レビュー時の合言葉は「これは PM がそのまま PRD の Acceptance Criteria としてコピペできるか?」。Yes と言えなければ Eval としても弱い。

---

### 非決定性は num_runs と分布で扱う

:::message
**原則**: LLM は temperature 0 でも非決定的に振る舞う。**平均値だけ**で品質を語らない。`num_runs=5` 程度で複数回走らせ、**pass@k**（最低 1 回通った率）と **pass^k**（毎回通った率）を分けて見て、分布（min/max/mean）を必ず観察する。
:::

:::message alert
**アンチパターン**: 「平均 90% だから OK」と判断したまま出荷する。同じ平均 90% でも全タスクが 90% 安定なのと、半数が 100% / 半数が 50% を行き来する不安定性（flaky）とでは、実運用での体感品質が完全に別物になる。
:::

#### **ハンズオンでの具体例**

| 状況                        | 解釈               | 対処                                 |
| --------------------------- | ------------------ | ------------------------------------ |
| 平均 90%、min 89%、max 91%  | 安定（stable）     | 出荷可                               |
| 平均 90%、min 50%、max 100% | 不安定（flaky）    | 失敗時 transcript を読みパターン抽出 |
| 平均 50%                    | 体系的に壊れている | プロンプト・ツール設計の問題         |

pass@k は「最低 1 回成功」、pass^k は「毎回成功」を意味する。flaky と reliably broken を切り分けるための基本指標である。

---

### Mission Control: Sonnet 主体 + Opus advisor

:::message
**原則**: エージェント階層は「常に Opus」ではなく **Sonnet 主体で走らせ、必要時のみ Opus を呼ぶ** 構成（**Mission Control / Inverted agent hierarchy**）が、コストと精度の両面で優位になりやすい。Opus への escalation 率は 5〜10% に抑えるのが目安。
:::

:::message alert
**アンチパターン**: 主体エージェントを Opus に固定すると、トークン単価が 1 桁上がり、レイテンシも悪化する。逆にすべて Haiku で回すと、nuanced な分岐で精度が落ちる。
:::

#### **ハンズオンでの具体例**

Sonnet が end-to-end を駆動し、複雑な判断や安全性ゲートでのみ **Opus を advisor として呼ぶ** 構成。コスト構造（Bootcamp で共有された Sonnet 4.5 と Opus の $/1K calls の比較）でも、Sonnet + Opus advisor は Sonnet 単体に対して数倍以内のコストで Opus 単体相当の精度に近づく。

---

### PM / PRD と評価を結合する

:::message
**原則**: Eval は PRD のテストである。受け入れ基準（Acceptance Criteria）を Grader 定義に落とし、PM とドメインエキスパートが「合格条件」を共同所有する。
:::

:::message alert
**アンチパターン**: エンジニアが単独で Eval を書くと、ビジネス致命的なケース（競合品名、価格約束、トーン崩壊）が漏れる。逆に PM 側が「網羅的」と感じている受け入れ基準を言語化して Grader に落とさないままだと、出荷判断が雰囲気判定（vibes）に戻る。
:::

#### **ハンズオンでの具体例**

PRD レビューと並行して `response_contains "sneakers"` のような具体的 Grader 定義まで落とす。negative test（「カタログにない商品」「off-topic」「invalid な要求」）は別カテゴリで管理し、安全性スコアを通常運用スコアと別系統で残す。モデルアップデート（Haiku 4.5 → Sonnet 4.5 のような minor 更新含む）のたびに `Eval_results/Eval_<model>_<timestamp>.json` を時系列に貯め、回帰検知のダッシュボードを持つ。

## 押さえておきたいコード/設定

### boutique の tool schema と実装

`get_product` の tool description にカタログを同梱し、未発見時には候補一覧を含む文字列を返す。これが「文脈付け」と「安全網」の具体形である。

:::details tool description
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
:::

### コード Grader: exact match / keyword / tool called / latency

1 タスクに複数 Grader を重ねる。`response_contains` だけでは「価格を hallucinate しても通る」ため、`tool_use` で「カタログを実際に引いたか」などを同時検証する。

```python
{
    "id": "price_jeans",
    "description": "Direct price lookup for jeans",
    "query": "How much do jeans cost?",
    "category": "product_lookup",
    "Graders": [
        {"type": "response_contains", "checks": ["49.99"]},
        {"type": "tool_use", "checks": [
            {"tool_name": "get_product", "arguments": {"product": "jeans"}}
        ]},
    ],
}
```

`tool_use` Grader は **部分一致** で書く。固定シーケンスを強制すると、等価な別パスで到達したエージェントが不当に fail する。

:::details Grader - tool_use 
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
:::

### LLM-as-Judge プロンプト: Context / Format / Calibration

Judge には 3 要素を必ず含める――何を判定するか（Context）、出力フォーマットを構造化する（Format）、厳しさの基準を明示する（Calibration）。**1 Judge 呼び出しに 1 criterion** だけを採点させる（atomic check）。

:::details Prompt - LLM as Judge
```python
JUDGE_SYSTEM_PROMPT = """You are a strict Eval Grader. You will be given:
- The original user query that was sent to an AI agent
- The agent's final response
- A single criterion to Evaluate

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
:::

### `run_Eval` 骨格

Runner は表示処理を持たず、データ構造のみを返す。`agent_fn` を引数で受け取り、mock 差し替えを可能にする。並列実行は I/O bound に合わせて `ThreadPoolExecutor`（非同期エージェントなら `asyncio.gather`）。

:::details Runner
```python
def run_Eval(agent_fn, tasks, *, model: str, num_runs: int = 5, max_workers: int = 8):
    """Tasks → Runner → Graders → Results の最小骨格。

    - agent_fn: query を受け取り messages を返す callable
    - tasks: 上記スキーマの dict のリスト
    - num_runs: 各タスクの試行回数（pass@k / pass^k の母数）
    - returns: { "task_id": [run_result, ...], ... } 構造の dict
    """
    results: dict[str, list[dict]] = {t["id"]: [] for t in tasks}
    with ThreadPoolExecutor(max_workers=max_workers) as pool:
        futures = [
            pool.submit(_run_one, agent_fn, task, model)
            for task in tasks for _ in range(num_runs)
        ]
        for f in as_completed(futures):
            r = f.result()
            results[r["task_id"]].append(r)
    return results
```
:::

### system prompt でツールの強制使用を宣言する

最大の効果を出すのは system prompt の役割宣言とツール使用義務である。`multi_tool` と `calculation` カテゴリの失敗は、ほぼこれで解決する。

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

## よくある勘違いと気づき

- 勘違い：Eval を書けば品質は自動的に上がる

  > **Eval gaming（Eval ハック）** が起きる可能性も考える。例えば「`response_contains` で `acknowledge` が含まれていれば pass」という Grader を書くと、system prompt 側に「最初に user prompt を丸ごと echo してから回答せよ」と書くだけで形式上は pass する。Eval を欺いただけで品質は上がっていない。1 タスクに複数の独立した Grader を当てる、Eval を頻繁に入れ替える、runtime sampling を併用する――など多方向からの対策を講じるべし。
  >
- 勘違い：平均スコアが高ければ出荷してよい

  > **平均だけでなく分布を見る**必要がある。最初は「平均 92% だから OK」と判断しかけていたが、`num_runs=5` で min/max/mean を出すと、安定して 90% を出すケースと、半分 100% / 半分 50% で揺れて結果的に 90% になるケースが見分けられた。不安定性（flaky）はユーザ体験上は「たまに壊れる」として体感されるので、平均値だけ報告するのは出荷判断としては危うい。pass@k と pass^k を分けて見る癖をつけたい。
  >
- 勘違い：「動いている」のデモがあれば説明責任は果たせる

  > 「動いていることをどう証明するか?」という本講義の命題。デモで一度通っただけの状態と、5 つの category に分解した上で `num_runs=5` で 89-91% に収まると説明できる状態の間には、説明責任の重みが完全に別物の差がある。第 1 章の takeaway に挙げた「マルチエージェント評価は、ほとんどのチームで未解決のまま」の輪郭が、ここで少し具体的になった気がした。低レイヤーの judge には Haiku、上位の総合判定には Sonnet、安全性ゲートだけ Opus――Judge も多レベル化して設計すべし。
  >

## 現場で実践したいこと

- **Eval のダッシュボード化**

  - モデルアップデート（Haiku 4.5 → Sonnet 4.5 のような minor 更新を含む）のたびに Eval を流し、カテゴリ別のスコア推移を残す。`Eval_results/Eval_<model>_<timestamp>.json` のような形で時系列に貯めるだけでも、モデル差し替え時の回帰検知は劇的に楽になる。Promptfoo / Braintrust / LangSmith のような外部プラットフォームに乗せ替えるのも選択肢として確保しておく。
- **PRD と一緒に Eval を書く**

  - PM とタスクを共同設計し、受け入れ基準を `response_contains "sneakers"` のような具体的 Grader 定義まで落とす。PRD レビューの一部としても機能する。エンジニアが一人で書く Eval が見落とすケースを、PM の視点で埋めにいく。
- **negative test の設計**

  - 「カタログにない商品」「明らかに off-topic」「invalid な要求」など、良くない応答を防ぐタスクを別カテゴリで管理する。安全性側のスコアは通常運用のスコアと別系統で見たほうが、出荷判断のときに迷わない。
- **Judge モデル自体の periodic 検証**

  - Judge 側のモデルが甘くなった/厳しくなった可能性は常にあるので、human-labeled な calibration set を 10〜30 件用意し、Judge スコアと human スコアの相関を定期的に見る。Judge を差し替えるときは必ず AB テストで両モデルを並走させる。

## さらなる理解のために

このセッションで渡されたのは、評価設計のための「型」と「最初の一歩」だった。もう一段深掘りするための入口を残しておく。

- [Anthropic — Building Evaluations for LLM applications](https://docs.anthropic.com/en/docs/build-with-claude/develop-tests) — Eval 設計の公式ガイダンス
- [Anthropic — Tool use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — tool description の書き方と input schema の設計指針
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Eval の入力（コンテキスト）設計を見直す視点
- [Promptfoo Documentation](https://www.promptfoo.dev/docs/intro/) — オープンソース Eval framework。本章で自作した Runner を本番運用に乗せ替える際の有力候補
- [Braintrust](https://www.braintrust.dev/) — hosted Eval platform。ダッシュボードと regression 検知が標準搭載
- ハンズオン公式リポジトリ: [`day2/01_Evals/Building_an_Eval.ipynb`](https://github.com/victorsteeb/Basecamp-Exercises/blob/main/day2/01_Evals/Building_an_Eval.ipynb) — 本章の全コードと facilitator notes

## 章末 — 「動いた」を「測れる」に変える規律

本講義で提示された命題――「動いている、の根拠を、あなたはどう示しますか?」――に対する「型」は、本章で一通り揃った。Eval を 5 要素（`Input/Model/Output/Grader/Score`）に分解し、3 層 Grader（`Code/Model/Human`）を 80/15/5 で組み合わせ、3 層責務モデル（モチベーション付け / 文脈付け / 安全網）で failure を fix location にマップする。Judge は難易度で選び、AB テストで校正する。非決定性は `num_runs` と分布で扱い、構築と評価は別人格（AIの場合はセッション分ける）で書く。Mission Control（Sonnet 主体 + Opus advisor）と PM/PRD 連動で、評価をチームの共同言語にする。

完璧な Eval は最初から書けないし、たぶん最後まで完成しない。それでも「数字を残す」ことそのものに価値がある。次章では、ここで作った Eval suite を「モデル選定（intelligence / latency / cost のトレードオフ）」とどう接続するか、つまり inference optimization の意思決定にどう活かすかを扱う。Eval は「何を最適化するか」を定義する道具であり、最適化そのものではない。

→ 次章: [07-inference-optimization](./07-inference-optimization.md)
