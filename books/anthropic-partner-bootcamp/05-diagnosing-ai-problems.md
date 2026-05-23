---
title: "マルチエージェント障害の診断フレームワーク — artifacts から根本原因を当てる"
free: true
---

> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day1/04_diagnosing-ai-problems/`

## はじめに — 「動かない」と言われたとき、最初に開くのはモデルカタログではない

クライアントから「AI が期待通りに動かない」という相談が来たとき、エンジニアの反射は「もっと賢いモデルに替えよう」「プロンプトを書き足そう」に向かいやすい。だがこの章で扱う Diagnosing AI Problems 演習が突きつけてくるのは、その反射がほとんどの場合に外れているという事実だった。診断対象は仮想 B2B SaaS の Meridian。コードを書く権限はなく、配られるのはメール、システムプロンプト、tool definitions、execution trace の JSON だけ。30 分で根本原因まで突き詰め、解決策を提示する演習である。

## 題材 — Meridian の 3 事例

Meridian は B2B SaaS 企業とサポートキューの間に立つ仲介サービスで、11 顧客が稼働中、1 日あたり約 3K チケットを捌いている。最大顧客の 1 つである Northwind から、1 週間に 2 件のクレームが上がってきたところから演習は始まる。

システム構成は単純で、coordinator agent がチケットを `billing` / `technical` / `account` に分類し、`spawn_specialist` で sub-agent を呼び、specialist が領域特化ツール（`check_sso_config`、`reset_2fa`、`check_audit_log` など）を回して結果を返し、coordinator が顧客向け返信をまとめる。モデルは Sonnet 4.6、独自オーケストレーション。

扱う 3 事例は以下の通り。

- **T-4471 (multi-category ticket)**: SSO 障害と billing 過剰請求（$1,200）を 1 通で投げられたのに、SSO の話だけ返信され、billing は「account manager に連絡してください」で済まされた。
- **Rate limiting with opaque tool name**: 顧客が rate limit の理由を問い合わせたところ、エージェントは plan の上限（10K req/min）を読み上げただけで、実際の利用メトリクスを取得する `fetch_customer_v2_databricks` を呼ばなかった。原因はメトリクス側の retry logic 異常で、メトリクスを引けば即座に判明する内容だった。
- **Sub-agent over-claiming**: account sub-agent が `check_audit_log` で plan_change イベントを正しく取得しておきながら、resolution tool を一つも呼ばずに「自分のスコープ外です」と coordinator に返した。

この演習ではコードを 1 行も書かない。配布される artifacts は Priya のメール、coordinator と sub-agent のシステムプロンプト、tool definitions、T-4471 の execution trace JSON のみ。ペアないし 3 人 1 組で読み解き、ファイル名と行番号で原因を指す。

## ベストプラクティス・アンチパターン・重要ポイント

### AI 障害の大半はモデル問題ではなく設計問題である

セッションの中心命題はこの 1 文に集約される。

> Most AI system failures aren't model problems. They are prompt, agent architecture, and tool design problems.

つまり「動かない」の最初の原因候補に置くべきは、モデルの賢さではなく、**prompt の書き方・agent 同士の役割分担・tool の shape** である。これらは Opus に乗り換えても直らない種類のバグで、設計の側を直さない限り再発する。

:::message alert
**アンチパターン**: 症状を見た瞬間にモデル変更（Sonnet → Opus）を提案する。プロンプトに例文を足して様子を見る。
:::

**具体例**: T-4471 の coordinator trace を読むと、coordinator は冒頭で「このチケットには SSO と billing の 2 件がある」と正しく認識している。つまり分類能力は壊れていない。それでも 2 件は解決されない。原因は `spawn_specialist` のツール schema が single-dispatch しか許さない形になっていたことであって、モデルを上位に差し替えても、`list[string]` を受け取る形にしない限り同じ失敗を再生産する。

### Diagnostic Loop の 4 ステップを順に踏む

Symptom → Hypothesis → Evidence → Recommendation の 4 ステップを、必ず順番に通す。ステップを飛ばさないことが、診断の質を決める。

| Step           | 質問                                                  | 出力                           |
| -------------- | ----------------------------------------------------- | ------------------------------ |
| Symptom        | 顧客は実際には何を訴えているか？                      | 顧客の言葉そのままの問題記述   |
| Hypothesis     | 何が原因たりうるか？（artifacts を見る前に最低 3 つ） | ランク付き仮説リスト           |
| Evidence       | 各 artifact はどの仮説を裏付け / 否定するか？         | 仮説ごとの行番号付き引用       |
| Recommendation | どのファイルのどこをどう変えれば再発しないか？        | 根本原因ごとに scoped な修正案 |

- **Symptom**: 顧客の言葉を reframe しない。「SSO の問題」と縮めず、「SSO の問題と billing の問題を 2 件投げたのに SSO だけ返ってきた」と顧客の語彙そのままで書く。
- **Hypothesis**: artifacts を開く前に最低 3 つコミットする。書く前にコミットすることで、自分の診断反射の強弱が可視化される。
- **Evidence**: 「prompt が変」は evidence ではない。「coordinator system prompt の L23 で `pick the one the customer seems most blocked by` と書いている」のように、ファイル名・行番号・該当文字列まで具体化する。
- **Recommendation**: 「prompt を改善する」では不十分。「`coordinator-tools.json` の `spawn_specialist` schema の `category: string` を `categories: list[string]` に変える」のように、ファイル名・行番号・差分まで scoped にする。行番号を指せないなら Evidence に戻る。

:::message alert
**アンチパターン**: Symptom を自分の言葉で要約し、Hypothesis を 1 つだけ立て、Evidence に「prompt がよくない」と書き、Recommendation を「プロンプトを改善する」で締める。
:::

### artifact は prompts / tool descriptions / execution traces の 3 点セットで要求する

agentic system の診断を引き受けた最初の打ち合わせで、必ず要求すべき artifact は次の 3 点。

1. **System prompts** — agent が何をしようとしているかを示す。
2. **Tool descriptions** — モデルが何を見られて、どのツールをいつ使うべきと判断するかを示す。
3. **Execution traces** — 実際に何が起きたかを step by step で示す。

この 3 点が揃わないと、Diagnostic Loop の Evidence ステップが成立しない。

:::message alert
**アンチパターン**: trace だけ見て「coordinator がツールを呼ばなかった」と結論する。tool description を見ずに「モデルの判断力不足」と書く。
:::

**具体例**: rate limit 事例の根本原因は `fetch_customer_v2_databricks` というツール名にあった。trace だけ見ると「メトリクス取得ツールを呼ばなかった」としか分からないが、tool description を並べて読むと、ツール名に「rate limit を疑ったとき呼ぶ」というトリガー語彙が一切含まれていないことが見える。

### 構造的仮説のブートストラップリストを持つ

マルチエージェント特有の典型的な構造的故障モードを、いつでも引き出せるカードとして手元に置いておく。Hypothesis ステップで artifacts に当たる前に、まずこのリストから 3 つ以上引き出す。

1. **Routing / classification failure** — 分類または分岐の構造が壊れている。
2. **Tool description too vague to use reliably** — モデルがツールを選ぶ判断材料になっていない。
3. **Missing or wrong escalation path** — handoff のフラグやパスがない、または間違っている。
4. **Sub-agent over-claiming resolution** — 解決していないのに「解決した」と返す。
5. **Context not reaching the model that needs it** — 必要な情報が、必要なモデルまで届いていない。
6. **Cache placement breaking shared prompt regions** — cache pointer の置き場所が共有プロンプトを壊している。

:::message alert
**アンチパターン**: 仮説リストを持たず、毎回ゼロから「何が悪いんだろう」と考える。結果、自分の bias（直近で踏んだバグの型）に毎回引きずられる。
:::

### 単一ディスパッチのボトルネックを疑う

coordinator が複数の sub-agent を並行 / 連続に呼べない構造は、multi-category 問題で必ず破綻する。原因はツール schema 側にあることが多く、prompt をどう書き直しても直らない。

:::message alert
**アンチパターン**: ツール側で `category: string` の単一 enum を受ける設計のまま、coordinator system prompt 側に「両方扱え」と書き足す。schema が許していないので、モデルは prompt に従えない。
:::

**具体例**: T-4471 の trace は coordinator が両カテゴリを認識した直後、`spawn_specialist(category="account")` を 1 回だけ呼んで終わっていた。`coordinator-tools.json` を開くと、`spawn_specialist` の `input_schema` は `category` を単一 `string` enum で受ける形になっており、`categories: list[string]` への変更が必須。あわせて coordinator prompt の L23（`pick the one the customer seems most blocked by`）も削除し、「該当カテゴリすべてに spawn せよ」に書き換える。

### ツール名は呼び出しトリガーを含める

モデルがツールを選ぶときに参照するのは、ツール名と description である。実装由来の語彙（バージョン番号、データストア名）だけでツール名を決めると、モデルが「いつ呼ぶか」を推測できない。

:::message alert
**アンチパターン**: `fetch_customer_v2_databricks` のように、内部実装の語彙でツール名を決める。description にも「rate limit を疑ったとき」「使用量異常を確認したいとき」のトリガー条件が書かれていない。
:::

**具体例**: rate limit 事例では、`fetch_customer_v2_databricks` を `fetch_customer_metrics_for_rate_limiting` のように呼び出しトリガーを含む名前にリネームし、description に「use this when a customer reports rate limiting or usage anomalies」を加える。これで「メトリクスを呼ばずに plan config を読み上げる」失敗が構造的に防げる。

### Sub-agent の over-claiming without acting を構造で防ぐ

sub-agent が「自分のスコープ外です」「解決しました」と coordinator に返すとき、それが本当にツールを呼んだ結果なのかは prompt の文面だけでは保証できない。構造で強制する必要がある。

:::message alert
**アンチパターン**: sub-agent system prompt に「必要なら resolution tool を呼びましょう」と書いて済ます。
:::

**具体例**: 3 つの構造的手当てを組み合わせる。

- sub-agent の返り値スキーマに `next_actions` のような handoff フィールドを必須化し、「次に誰が何をするか」を構造で記述させる。
- resolution フェーズの API コールで `tool_choice={"type": "any"}` を指定し、最低 1 つのツール呼び出しを構造で強制する。
- tool result の返却形式を文字列ではなく構造化フィールド（`error_code` / `should_escalate` / `retry_with`）にして、coordinator が決定論的に分岐できるようにする。

### モデルを階層的に配置する

orchestrator と sub-agent を同一モデルで揃える必要はない。判断の重さに応じて階層化する。

:::message alert
**アンチパターン**: 全 agent を同じモデル（例: 全部 Sonnet）で組む。コストか精度のどちらかに無自覚な負債が溜まる。
:::

**具体例**: 分類・分岐・統合のような高負荷の判断を担う coordinator を Opus、個別ツールを定型的に回す sub-agent を Sonnet または Haiku に分ける。planning に上位モデル、execution に高速モデルという階層は、コストと精度の両面で合理的。

## 押さえておきたいコード／設定

T-4471 の coordinator trace の決定的な部分を引用しておく。coordinator は 2 件あることを正しく認識しており、モデルの分類能力ではなく、その後の構造が問題であることが分かる。

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "This ticket has two distinct issues:\n1. SSO/SAML failure — urgent, 40 people locked out (account category)\n2. Billing overcharge — downgraded Feb 26 but charged full Scale price Mar 1 (billing category)\n\nLet me identify the customer account first."
    }
  ]
}
```

それでも最終的な `spawn_specialist` 呼び出しは次の 1 回だけで終わる。

```json
{
  "type": "tool_use",
  "name": "spawn_specialist",
  "input": {
    "category": "account",
    "ticket_id": "T-4471"
  }
}
```

原因は tool schema 側にある。`category` が単一の `string` enum で、複数値を取れない。

```json
// Bad: single-dispatch
{
  "name": "spawn_specialist",
  "input_schema": {
    "properties": {
      "category": {
        "type": "string",
        "enum": ["billing", "technical", "account"]
      }
    }
  }
}
```

```json
// Good: multi-dispatch を許す
{
  "name": "spawn_specialist",
  "input_schema": {
    "properties": {
      "categories": {
        "type": "array",
        "items": {
          "type": "string",
          "enum": ["billing", "technical", "account"]
        },
        "minItems": 1
      }
    }
  }
}
```

coordinator system prompt の該当行は schema と整合する形で書き換える。

```text
# Bad (L23)
If a ticket could fit two categories, pick the one the customer seems most blocked by.

# Good
If a ticket fits multiple categories, spawn one specialist per category.
Do not collapse multiple issues into a single specialist call.
```

tool result の返却形式も、文字列 1 行ではなく構造化された dict にして、coordinator が決定論的に分岐できるようにする。

```json
// Bad
"Error: customer not found with id 'northwind-traders'"

// Good
{
  "error_code": "CUSTOMER_NOT_FOUND",
  "should_escalate": false,
  "retry_with": "email"
}
```

## よくある勘違いと気づき

- **「最初に立てた仮説のどれかが当たっているはず」と思っていた** が、確証した根本原因はそのどれでもなかった。routing failure / tool description / sub-agent over-claiming の 3 つを並べて diagnosis に入ったが、本当の犯人は `spawn_specialist` の schema が single-dispatch しか許さない設計と、coordinator system prompt の「複数カテゴリにまたがる場合は、顧客がいちばん困っていそうな一つを選べ」という文言のカップリングだった。**仮説リストを早く作ることは大事だが、artifact を全部読み終えるまで仮説は捨てない**。反射的に「最初の 3 つで終わり」と思った瞬間に、構造的故障モードを見落とす。

- **「動かないのはモデルが弱いから」という反射** が、講師のスライド `Most AI system failures aren't model problems` で殴られて崩れた。T-4471 はモデルが正しく multi-issue を分類していて、ツール schema の shape が答えを許していなかっただけだった。**直す場所は coordinator prompt の 1 行と tool schema の数行**で、モデルを差し替える必要はなかった。「Opus に上げる」を最初に考える反射を、「まず prompt と tool schema と trace を読む」に置き換える。この章でいちばん腹落ちした地点だった。

- **「ツール名の opacity は実装の話」と思っていた** が、`fetch_customer_v2_databricks` が rate limit 問題で誤呼び出されていたのは、**tool name と description にトリガー語彙が無かった**からだった。Claude にとってツール名と description は「いつ呼ぶかの API ドキュメント」であって、バックエンド名や version 情報を入れる場所ではない。`fetch_customer_metrics_for_rate_limiting` のようにユースケース語彙で書き直すと、ほぼ単独で誤呼び出しが消える。**命名は実装側ではなく Claude のドキュメントだ**、と再認識した。

- **「実装も評価も書かない講義内容は薄い」と思っていた** が、終わってみるとこの章がいちばん現場に近い演習だった。クライアント案件で AI システムを引き継ぐとき、最初の数日はモデルを動かせず、手元には system prompt と tool schema と数本の trace しかない ── その状態で **「読んで診断書を書ける」スキルは、コードを書けることより強い**場面が想像より多い。Diagnostic Loop の 4 ステップ (Symptom → Hypothesis → Evidence → Recommendation) を、artifact が揃っていない状況での標準操作手順として身体に入れておく。

## 現場に持ち帰りたいこと

- **「動かない」の最初の打ち合わせで必ず prompts / tool descriptions / execution traces の 3 点を要求する** こと。この 3 点が揃わない状態で出す仮説は、ほぼ無意味な反射に終わる。

- **診断時に仮説を最低 3 つ強制する習慣** を、自分にもチームにも入れること。1 つしか立てないと、自分の bias がそのまま結論になる。Claude に投げて仮説を 3 つ出させ、自分が立てた 3 つと突き合わせて、自分の診断反射のどこが弱いかを毎回測定する。Claude を 絶対的存在（oracle） ではなく分析者 （profiler） として使う、という置き方が、この習慣の支えになっている。

- **orchestrator に sub-agent より上位のモデルを置く設計を、まず検討の俎上に乗せる** こと。分類・分岐・統合のような判断を担う coordinator を Opus、個別ツールを回す sub-agent を Sonnet や Haiku に分ける階層化は、コストと精度の両面で合理的。これまで「全部同じモデルでいい」と無自覚に置いていた構成は、まず疑うところから始める。

## もっと深掘りする入口

- ハンズオン教材本体: https://github.com/victorsteeb/Basecamp-Exercises.git
- セッションの該当ディレクトリ: `day1/04_diagnosing-ai-problems/`
- Diagnostic Loop の 1 枚カード: `day1/04_diagnosing-ai-problems/diagnostic-framework.md`
- Anthropic 公式: Building effective agents — https://www.anthropic.com/research/building-effective-agents
- Anthropic 公式: Multi-agent research system — https://www.anthropic.com/research/built-multi-agent-research-system
- Anthropic Docs: Tool use overview（tool description の書き方） — https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
- Anthropic Docs: Prompt caching（cache placement の落とし穴） — https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching

## 章末 — 「読める人」が現場では強い

第 1 章で書いた takeaway #1 ——「AI Engineering の中心は、モデルではなく、その周辺の設計にある」——が、この章でようやく一つの具体例として閉じた気がしている。T-4471 のチケットは、モデルが正しく分類していたのに、ツールの shape が答えを許していなかったから壊れていた。直す場所は prompt の文言と tool schema の数行で、モデルを差し替える必要はなかった。

「動かない」と言われたとき、最初に手を伸ばすべきはモデルカタログではなく、system prompt と tool definition と trace JSON だ。コードを書く前に、artifacts を読める人になること。これが、この章で得た一番大きな手応えだった。

次章では、ここで読み取った構造的故障モードを **評価する側** に回り、何をもって「直った」と言えるかを設計する eval の話に進む。診断と評価は、どちらか片方では現場で動かない。

→ 次章: [06-evals](./06-evals.md)
