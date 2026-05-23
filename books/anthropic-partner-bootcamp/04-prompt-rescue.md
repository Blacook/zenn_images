---
title: "壊れたプロンプトを救う — Eval 駆動のプロンプトエンジニアリング"
free: true
---

> ハンズオン公式リポジトリ: https://github.com/victorsteeb/Basecamp-Exercises.git
> 該当ディレクトリ: `day1/03_prompt-rescue/`

## 21 件のチケットが、最初の一発で半分しか通らなかった話

ノートブックを開いて、用意された eval ハーネスをまず素のまま実行した。出てきたのは、21 件中 10 件パス、というスコアだった。POC のデモでは普通に動いていたはずのプロンプトが、本番想定のチケットを通した瞬間、半分以上で外れる。優先度は「URGENT」の一言に引きずられ、エンティティ欄には存在しない製品名が混じり、後段で受けるはずだった JSON は途中で壊れていた。

最初に感じたのは「プロンプトが下手だったのか?」という素朴な疑問だ。だが画面に並んだ失敗ケースを 1 件ずつ眺めていくと、それは違うことが分かってきた。プロンプトを「上手く書く」ことに本質はない。失敗の型を見つけ、仮説を立て、修正し、もう一度回す。その反復ループそのものが、プロンプトエンジニアリングの正体だった。

この章はその実感を中心に据えた、Eval 駆動の救出記録である。

## 題材

題材は、サポートチケット分類の本番プロンプトを修復するというシナリオだ。優先度 (P1〜P4)、エンティティ抽出 (製品名・バージョン・影響ユーザ数など)、初期応答ドラフトの 3 つを、1 回の Messages API 呼び出しで返す設計。POC のデモではきれいなチケットだけで評価していたため通っていたが、本番では英語非ネイティブ、複数問題の混在、トーンだけ強い機能要望といった「現実のチケット」が一斉に流れ込んでくる。

ハンズオンで与えられるのは、21 件のテストケース (clean / multi-issue / vague / non-native / feature-request / complex の 6 カテゴリ) を持つ eval ハーネス。モデルは `haiku`、応答時間は 5 秒以内、という現実的な制約付きで、ひたすらこの 21 件を回す。

## 5 つの典型失敗パターンを目撃する

ベースラインプロンプトを回した結果を、カテゴリ別スコアと失敗ログでざっと俯瞰すると、失敗の系統は驚くほどはっきり 5 つに分かれていた。

ひとつめは、**トーンによる優先度誤判定**。"CRITICAL: No SSO support" と書かれたチケットを、内容は機能要望 (P4) なのに P1 と分類してしまう。緊急性を煽る語彙だけで判断していて、本文に「これがないと困る」しか書いていない事実を読み取れていない。

ふたつめは、**エンティティの捏造**。"things aren't working right" という曖昧なチケットから、製品名 `PageLoader` とバージョン `2.1` が抽出される。"several people are affected" から `affected_users: 5` というそれらしい数字が出てくる。本文のどこにも書かれていないはずの情報が、JSON のフィールドを埋めるために生み出されていた。

みっつめは、**JSON 破綻**。ベースラインには "Always include all JSON fields even if empty" と書いてある。にもかかわらず `null` と空配列と空文字列のどれを使うかが曖昧で、後段の Zendesk 連携が parse error で落ちる。

よっつめは、**多問題チケットの統合化**。「ログインも、レポートも、メール通知も壊れている」というチケットが、1 つの優先度と 1 つの応答にまとめられてしまう。本来であれば 3 件の独立した issue として扱うべきものが、強引に一本化されている。

いつつめは、**機能要望への修正約束**。P4 の機能要望に対して "We'll fix this right away" と返してしまう。SLA を守るどころか、現場では「言質」として持ち出される最悪のパターンだ。

5 つを並べてみて気づくのは、どれも「モデルが弱いから」起きているわけではないということだ。プロンプトが、その判断ロジックを書ききれていない。それだけだった。

## 直していくうちに浮かび上がった 12 の原則

失敗ケースを 1 つずつ潰していく過程で、自然と自分の中にチェックリストが溜まっていった。それを後から整理したのが以下の 12 個だ。「これを最初から守れば完璧」ではなく、「Eval が失敗を返すたびに、このどれかに該当する」という形で機能する。

### 1. 明確で具体的な判定基準

「優先度を判定して」ではなく「P1 = 全ユーザ影響のシステム停止、P2 = 主要機能の障害かつ多数ユーザ影響、…」のように判断ロジックまで踏み込む。判定は **内容ベース** であり **トーンベースではない** ことを明記する。

### 2. XML / Markdown による構造化

Claude は XML タグを「指示の境界」として強く認識する。`<task>`、`<priority_rules>`、`<input>` のように役割ごとに区切ると、ルールと入力データが混ざらない。

### 3. アンチパターンの Bad / Good 対比

LLM は単純な否定形 ("〜しないでください") を見落としやすい一方、`<bad>...</bad>` と `<good>...</good>` を理由付きで並べる対比構造には強く反応する。失敗例 → 望ましい挙動 → なぜ、の三点セットで書くと従われやすい。

### 4. ハルシネーション抑制

「明示されたエンティティのみ抽出する。記載がないフィールドは `null` にする。推測・補完は禁止」と挙動を明示する。沈黙は禁止ではない、という前提でモデルは動くので、`null` という具体的な逃げ道を与えるのが要点。

### 5. confidence の出力

情報量に応じた `high / medium / low` を出させ、後段でしきい値で振り分ける。製品名・バージョン・エラーコードが揃えば high、感情的訴えだけなら low + 追加情報要求、というポリシーを書く。

### 6. ロール・ペルソナの定義

「あなたは冷静なシニアサポートエンジニアです。感情に左右されず、内容で判断します」のような役割定義は、トーンに引きずられた誤分類への強力な防波堤になる。

### 7. Few-shot プロンプティング

入出力例を 2〜3 個、特に **エッジケースを含めて** 示す。多問題チケット、曖昧チケット、激怒チケットの例は効果が高い。

### 8. Chain of Thought

「出力前に以下を順に確認せよ」と確認ステップを明示する。実質的な業務影響 → 緊急性語彙とのギャップ → 明示エンティティの列挙 → 不明項目の null 化 → JSON 構造の完全性、という順序が定番。

### 9. JSON スキーマの厳格化

`response_format` / `output_config` の JSON schema 機能を使い、`additionalProperties: false` を効かせ、フィールド欠落・追加を禁止する。"JSON で返して" ではなく、スキーマと許可される enum 値まで指定する。

### 10. コンテキスト順序（sandwich pattern）

「システム指示 → ルール → 例 → 入力 → 重要指示の再掲」の順で並べる。Claude は冒頭と末尾の指示に従いやすいため、最重要ルールは末尾にも再掲する。

### 11. Eval 駆動の反復改善

ベースライン測定 → 失敗カテゴリ分析 → 仮説立案 → 修正 → 再測定 → 差分検証、というループを回す。**1 回で完璧を狙わない** のが最重要原則。

### 12. プロンプトインジェクション対策

ユーザ入力は必ず `<user_input>` などのタグで囲み、システム指示と物理的に分離する。タグ内のテキストは「データ」であり「指示」ではない、という規約を明示する。

## 前提が崩れた瞬間

12 個の原則のうち、自分の中で一番ひっくり返ったのは 2 つだ。

ひとつは、**否定形「〜するな」で抑止できると思っていた** こと。"製品名を捏造しないでください" と書けば伝わると思っていたが、実際にはモデルは否定形をあっさり見落とす。Bad / Good 対比に書き換え、それぞれに `<reason>` を添えた途端、捏造系の失敗が一気に減った。「禁止」を書くより、「失敗例と望ましい例と、その理由」を並べるほうが、構造として強い。

もうひとつは、**1 ショットで完璧を目指せると思っていた** こと。ある参加者は、20/21 まで通った後に最後の 1 件に固執して全体を書き換え、スコアが 12/21 まで逆戻りしたという。話を聞きながら、これは自分でも同じことをやりかけたなと感じた。カテゴリ別スコアを見て、最も弱い 1 カテゴリだけを直す。clean を壊さずに feature-request を上げる。次の反復で vague を狙う。そういう制約付きの反復のほうが、結果的に早く 21/21 へ到達した。

「うまく書く」のではなく、「測りながら直す」。プロンプトエンジニアリングが工学である、という言葉の意味が、このときようやく腹落ちした。

## 押さえておきたいコード — XML タグ付きプロンプトの Bad / Good 対比

API キーや実機の出力は載せず、構造に集中する。Bad / Good 対比を 2 つ示す。

### 例1: 優先度判定ルール

```xml
<!-- Bad: 1 行ルール、トーン引きずられに脆弱 -->
<rule>
  Set priority P1 if the ticket is urgent.
</rule>
```

```xml
<!-- Good: 内容ベース、Bad/Good 対比、理由付き -->
<priority_rules>
  <p1>
    <criteria>System down AND affects all users</criteria>
    <example>Production API returning 500 for every request</example>
  </p1>
  <p2>
    <criteria>Major feature broken AND affects many users</criteria>
  </p2>
  <p3>
    <criteria>Workaround exists OR affects single user</criteria>
  </p3>
  <p4>
    <criteria>Feature request or cosmetic issue</criteria>
  </p4>

  <anti_pattern>
    <bad>Classify "URGENT!! No SSO" as P1 because of the word URGENT</bad>
    <good>Classify "URGENT!! No SSO" as P4 — it is a feature request regardless of tone</good>
    <reason>Urgency vocabulary is not the same as business impact</reason>
  </anti_pattern>
</priority_rules>
```

### 例2: エンティティ抽出のハルシネーション抑制

```xml
<!-- Bad: 補完される余地が大きい -->
<task>Extract product, version, and affected users from the ticket.</task>
```

```xml
<!-- Good: null 規約と Bad/Good 例を明示 -->
<entities_rules>
  <policy>
    Only extract entities that are explicitly stated.
    Missing fields MUST be null. Never guess, never infer.
  </policy>

  <anti_pattern>
    <bad>
      Input: "things aren't working right"
      Output: {"product": "PageLoader", "version": "2.1"}
    </bad>
    <good>
      Input: "things aren't working right"
      Output: {"product": null, "version": null}
    </good>
    <reason>No product or version was mentioned in the ticket</reason>
  </anti_pattern>

  <anti_pattern>
    <bad>
      Input: "several people are affected"
      Output: {"affected_users": 5}
    </bad>
    <good>
      Input: "several people are affected"
      Output: {"affected_users": null}
    </good>
    <reason>"several" is not a specific count</reason>
  </anti_pattern>
</entities_rules>

<user_input>{{ticket}}</user_input>

<final_reminder>
  Before responding, verify:
  1. Are all extracted fields explicitly present in the ticket?
  2. Are unknown fields set to null (not "", not 0, not "unknown")?
  3. Does the JSON match the schema exactly?
</final_reminder>
```

`<user_input>` で指示と入力を物理分離し、末尾に `<final_reminder>` を置く sandwich pattern。短いがこれだけで JSON 破綻と捏造の両方が体感で半分以下に落ちる。

## 現場に持ち帰りたいこと

ハンズオンを抜けてから振り返ったときに、これは現場でもそのまま使えるな、と思った観点が 3 つある。

ひとつめは、**prompt chaining による分割**。優先度・エンティティ・応答を 1 プロンプトで処理していたものを、`priority` 分類 → `entities` 抽出 → `response` 生成、という 3 ステップに分けると、各ステップが短くなり、Few-shot も対象を絞れて精度が上がる。どこで壊れたかが特定しやすいし、ステップごとに `haiku` と `sonnet` を使い分ける余地も生まれる。「1 プロンプトで全部やろうとしない」という設計判断は、構造化と並ぶ最重要のレッスンだった。

ふたつめは、**Eval ハーネスをまず用意する**。プロンプトを書く前にテストケースを用意し、カテゴリ別にスコアを取れる状態を先に作る。これがないと、修正が本当に効いたのかが感覚でしか分からない。後段で起きた事故が、プロンプトのどの変更に起因するのかも追えなくなる。

みっつめは、**Confidence を必ず出力させる**。`high / medium / low` を返させ、low のときは追加情報要求に分岐する。これは単に分類精度を上げるだけでなく、「分からないものを分からないと言える」モデルにするための入口になる。後段ワークフローの設計が、確信度ベースで一段クリーンになる。

加えて、現実的な制約として `haiku` で 5 秒以内、という条件を意識しつづけたのも良い経験だった。Chain of Thought を冗長にし過ぎる、Few-shot を 10 件以上入れる、といった「品質は上がるが遅くなる」改善は、本番制約を破る。examples は 2〜3 件、CoT のステップは 4〜5 個まで、というあたりが現実解だ。

## もっと深掘りする入口

- [Anthropic Docs — Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Anthropic Docs — Use XML tags to structure your prompts](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags)
- [Anthropic Docs — Be clear, direct, and detailed](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-direct)
- [Anthropic Docs — Use examples (multishot prompting)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting)
- [Anthropic Docs — Let Claude think (chain of thought)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought)
- [Anthropic Docs — Chain complex prompts](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-prompts)
- [Anthropic Docs — Reduce hallucinations](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)
- [Anthropic Docs — Mitigate jailbreaks and prompt injections](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)
- [Anthropic Docs — Create strong empirical evaluations](https://docs.anthropic.com/en/docs/test-and-evaluate/develop-tests)
- [Anthropic Cookbook — Prompt engineering examples](https://github.com/anthropics/anthropic-cookbook)
- [ハンズオン公式リポジトリ](https://github.com/victorsteeb/Basecamp-Exercises.git) (`day1/03_prompt-rescue/`)

## おわりに — Eval なしでプロンプトを改善するのは、祈祷だ

第 1 章の冒頭で書いた takeaway のひとつ、「AI システムの失敗のほとんどはモデルではなく、プロンプト・エージェント設計・ツール設計に起因する」という観察は、この章でようやく自分の手触りに変わった。21 件のチケットをひたすら回し、失敗ログを眺め、仮説を立てて 1 行直し、再測定する。その繰り返しを 10 周もすると、プロンプトが「文章」ではなく「テストで担保された仕様」のように見え始める。

Eval を持たずにプロンプトを直し続けるのは、結局のところ祈祷だ。書き換えるたびに「良くなった気がする」だけで、本当に良くなったかはわからない。逆に、カテゴリ別スコアと失敗ログをセットで持っていれば、プロンプトは少しずつ、確実に良くなる。

次章では、プロンプト単体ではなく **エージェントが複数ステップで動いたときに何が壊れるのか** を扱う。プロンプトを救ったあとに待っているのは、システムプロンプト・ツール記述・実行トレースが絡む、もう一段大きな問題空間だ。

→ 次章: [05-diagnosing-ai-problems](./05-diagnosing-ai-problems)
