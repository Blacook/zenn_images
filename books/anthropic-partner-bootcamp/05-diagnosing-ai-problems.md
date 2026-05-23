---
title: "マルチエージェント障害の診断フレームワーク — artifacts から根本原因を当てる"
free: true
---

> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day1/04_diagnosing-ai-problems/`

## メールを読み終えた瞬間に立てた仮説は、ほぼ全部外れていた

クライアント Priya からのメールには、こう書いてあった。チケット T-4471 で SSO の障害と billing の過剰請求の 2 件を同時に投げたのに、AI サポートシステムは SSO の話しか返してこなかった。billing については「account manager に連絡してください」と書いてあった、と。

最初に頭をよぎったのは「分類モデルの精度が落ちているのではないか」「sub-agent のプロンプトが billing の解釈に失敗しているのではないか」「モデルを Sonnet から Opus に変えれば直るのではないか」という、ありふれた仮説たちだった。コードを動かす権限はなく、artifacts も開いていない段階で、こちらの脳はすでに「モデルが弱いから」という結論に向かって走り始めていた。

最後に確証した根本原因は、そのどれでもなかった。`spawn_specialist` というツールの schema が `category: string` の単一 enum でしか sub-agent を呼べない設計になっていて、しかも coordinator のシステムプロンプトにも「複数カテゴリにまたがる場合は、顧客がいちばん困っていそうな一つを選べ」と書かれていた。要するに、構造的に single-dispatch しか許していなかった。これに気づいたとき、自分が立てた最初の三つの仮説がいかに反射的だったかを思い知った。

## 題材 — コードを 1 行も書かずに、artifacts だけで診断する

舞台は仮想 B2B SaaS 企業 Meridian のサポートチケット triage システム。coordinator agent がチケットを読んで `billing` / `technical` / `account` を分類し、`spawn_specialist` で sub-agent を呼び、最後に顧客向け返信をまとめる。sub-agent 側は `check_sso_config`、`reset_2fa`、`check_audit_log` のような領域特化のツールを持っている。

このセッションでは、コードを実行できない。配られるのは Priya のメール、coordinator と sub-agent のシステムプロンプト、tool definitions、そして T-4471 の execution trace JSON だけだ。30 分の制限時間で、ペアないし 3 人 1 組で artifacts を読み、行番号付きの根本原因を当てに行く。

割り切りが大胆だと感じた。実装も評価も書かない。「読む」だけのトレーニング。けれど終わってみると、これがいちばん現場に近い演習だった気がしている。クライアント案件で AI システムを引き継ぐとき、最初の数日はモデルを動かせず、手元には system prompt と tool schema と数本の trace しかない、という状況がふつうにある。「読める」ことが武器になる場面は、想像より多い。

## Diagnostic Loop の 4 ステップを、エッセイとして書き直す

ここで導入されたのが **Diagnostic Loop** という 4 ステップの型だった。Symptom → Hypothesis → Evidence → Recommendation。並べてしまえば普通だが、運用してみると一つひとつに罠があった。

**Symptom** では、顧客の言葉を reframe しないことが求められた。「SSO の問題」と縮めず、「SSO の問題と billing の問題を 2 件投げたのに SSO だけ返ってきた」と顧客の語彙そのままで書く。後から「single dispatch しかしていない」という構造的仮説に辿り着けたのは、ここで「2 件」という顧客の言葉を捨てなかったからだった。

**Hypothesis** で痛感したのは、artifacts を開く前に最低 3 つ仮説を書け、というルールの厳しさだった。1 つだと自分の bias（自分の場合は「prompt が悪い」に偏っていた）に引きずられる。マルチエージェント特有の典型的な構造的故障モードを最初にカードとして持っておくと、ここの解像度が一気に上がる。routing / classification failure、opaque な tool description、sub-agent の over-claiming、handoff フラグの欠落、必要なコンテキストがモデルまで届いていない、cache placement の事故。この六つを、いつでも引き出せるリストとして持っておくべきだ、と腹に落ちた。

**Evidence** のステップでは「prompt が変」は evidence ではない、と何度も突き返された。evidence とは「coordinator system prompt の L23 で `If a ticket could fit two categories, pick the one the customer seems most blocked by` と書いている」という粒度の引用のことだ。ファイル名、行番号、該当文字列。ここまで具体化しないと、後の Recommendation がふわっとした「prompt を改善する」で止まる。

**Recommendation** は「どのファイルの、どの行を、どう変えるか」まで落とす。`coordinator-tools.json` の `spawn_specialist` の input schema を `category: string` から `categories: list[string]` に変える、coordinator system prompt の L23 を削除して「該当カテゴリすべてに spawn せよ」と書き換える、というレベルまで。行番号を指せないなら Evidence に戻る、という単純なルールが、診断品質を決定的に持ち上げた。

## 「動かないのはモデルが弱いから」という反射が崩れた瞬間

セッションの結論メッセージはこう書かれていた。

> Most AI system failures aren't model problems. They are prompt, agent architecture, and tool design problems.

この一文の重みを、artifacts を読み終えてから読み返した瞬間にようやく実感した。T-4471 の coordinator trace を開いたとき、coordinator は冒頭で「このチケットには SSO と billing の 2 件がある」と正しく認識していた。モデルの分類能力には問題がなかった。それでも 2 件は解決されなかった。理由は、`spawn_specialist` が一度に 1 カテゴリしか取れない設計だったからだ。

つまり、モデルは合っているのに、ツールの shape が答えを許していなかった。これは「Opus に上げれば直る問題」ではない。`list[string]` を受け取る形にしない限り、どんなモデルでも同じ失敗をする。AI Engineer の仕事の中心が「モデルを乗り換える」ことではなく「system prompt と tool definition と trace を読み解く読解力」だ、という言葉が、ここで完全に腹落ちした。

似たことが、account sub-agent の trace でも起きていた。`check_audit_log` で plan_change イベントを正しく取得しておきながら、resolution tool を一つも呼ばずに「自分のスコープ外です」と coordinator に返していた。over-claiming with under-acting と呼ばれるアンチパターンだ。これも、より賢いモデルに変えても直らない。「sub-agent の返り値に `next_actions` のような handoff フィールドを構造的に持たせる」「resolution フェーズで `tool_choice={"type": "any"}` を使って最低 1 ツール呼び出しを構造で強制する」という、設計の側の手当てが必要になる。

## 押さえておきたい artifacts の引用

T-4471 の coordinator trace の決定的な部分を引用しておく。これだけで複数の故障モードが裏付けられる。

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

coordinator は **2 件あることを正しく認識している**。にもかかわらず、最終的な `spawn_specialist` 呼び出しは次の 1 回だけだった。

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

これが **routing failure** の現場証拠になる。原因は `coordinator-tools.json` の `spawn_specialist` schema を見ればすぐ分かった。

```json
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

`category` が単一の `string` enum で、しかも coordinator system prompt の L23 にはこう書かれている。

> If a ticket could fit two categories, pick the one the customer seems most blocked by.

ツール schema が single-dispatch しか許さず、prompt も single-dispatch を前提に書かれている。複数カテゴリの問題は、この設計のままでは構造的に解決できない。

加えて、tool result の返し方にも学びがあった。今回の trace では `"Error: customer not found with id 'northwind-traders'"` のような文字列 1 行で返っていたが、これだと coordinator は「escalate するか continue するか」をプロンプト依存で判断するしかない。次のような構造化フィールドを返すように設計しておけば、決定論的に分岐できる。

```json
{
  "error_code": "CUSTOMER_NOT_FOUND",
  "should_escalate": false,
  "retry_with": "email"
}
```

Diagnostic Loop の 4 ステップは、表として手元に置いておくと現場で使いやすい。

| Step | 質問 | 出力 |
|------|------|------|
| Symptom | 顧客は実際には何を訴えているか？ | 顧客の言葉そのままの問題記述 |
| Hypothesis | 何が原因たりうるか？（artifacts を見る前に最低 3 つ） | ランク付き仮説リスト |
| Evidence | 各 artifact はどの仮説を裏付け / 否定するか？ | 仮説ごとの行番号付き引用 |
| Recommendation | どのファイルのどこをどう変えれば再発しないか？ | 根本原因ごとに scoped な修正案 |

## 現場に持ち帰りたいこと

このセッションから持ち帰った行動指針は二つに絞っている。

一つは、**orchestrator には sub-agent より上位のモデルを置く設計を、まず検討の俎上に乗せる**こと。分類・分岐・統合のような判断を担う coordinator を Opus、個別ツールを回す sub-agent を Sonnet や Haiku に分ける、という階層化はコストと精度の両面で合理的だ。これまで「全部同じモデルでいい」と無自覚に置いていた構成は、まず疑うところから始める。

もう一つは、**診断時に仮説を最低 3 つ強制する習慣**を、自分にもチームにも入れること。1 つしか立てないと、自分の bias がそのまま結論になる。Claude に投げて仮説を 3 つ出させ、自分が立てた 3 つと突き合わせて、自分の診断反射のどこが弱いかを毎回測定する。Claude を oracle ではなく profiler として使う、というフレーズが、この習慣の支えになっている。

## もっと深掘りする入口

- ハンズオン教材本体: https://github.com/victorsteeb/Basecamp-Exercises.git
- セッションの該当ディレクトリ: `day1/04_diagnosing-ai-problems/`
- Diagnostic Loop の 1 枚カード: `day1/04_diagnosing-ai-problems/diagnostic-framework.md`
- Anthropic 公式: Building effective agents — https://www.anthropic.com/research/building-effective-agents
- Anthropic 公式: Multi-agent research system — https://www.anthropic.com/research/built-multi-agent-research-system
- Anthropic Docs: Tool use overview（tool description の書き方） — https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
- Anthropic Docs: Prompt caching（cache placement の落とし穴） — https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching

## 章末 reflection — 「読める人」が現場では強い

第1章で書いた takeaway #1 ——「AI Engineering の中心は、モデルではなく、その周辺の設計にある」——が、この章でようやく一つの具体例として閉じた気がしている。T-4471 のチケットは、モデルが正しく分類していたのに、ツールの shape が答えを許していなかったから壊れていた。直す場所は prompt の文言と tool schema の数行で、モデルを差し替える必要はなかった。

「動かない」と言われたとき、最初に手を伸ばすべきはモデルカタログではなく、system prompt と tool definition と trace JSON だ。コードを書く前に、artifacts を読める人になること。これが、この章で得た一番大きな手応えだった。

次章では、ここで読み取った構造的故障モードを **評価する側** に回り、何をもって「直った」と言えるかを設計する eval の話に進む。診断と評価は同じコインの裏表で、どちらか片方では現場で動かない。

→ [第6章 評価駆動でモデルとプロンプトを比較する — Building an Eval](06-evals)
