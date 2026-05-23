---
title: "Context Engineering — Context Rot との戦い"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day2/03_context-engineering/`
> **参考研究**:
> - Chroma's *Context Rot: How Increasing Input Tokens Impacts LLM Performance*: https://www.trychroma.com/research/context-rot
> - Anthropic *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

---

## 自分の前提が崩れた 2 日目の午後

SF の 2 日目、Context Engineering のセッションが始まったとき、自分はわりと余裕で椅子に座っていた。Prompt の書き方ならそこそこ自信があったし、1M token の context window を持つモデルも触ったことがある。「長い入力に強くなったんでしょ、知ってる」くらいの心持ちだった。

その心持ちは、最初の実験パートで完全に折れた。`apple` をひたすら並べた中に 1 つだけ `apples` を混ぜ、「そのまま再現せよ」とだけ命じる、推論を一切要しない単純な複製タスク。語数を増やしていくと、ある地点からモデルは普通に「直して」しまう。文字列を写すだけのタスクですら、入力が長くなれば崩れていく — その挙動を目の前のグラフで見た瞬間、「とりあえず context に全部入れておけば良い」という自分の癖が、研修の中でいちばん深い場所で殴られた。

本書を通してずっと著者が言いたかった「Context Engineering は nice-to-have ではなく規律だ」という第1章 takeaway #2 は、たぶんこの章で回収される。**個人的に SF で一番刺さったセッション**を、ここに書き残す。

---

## 題材 — 3 つの実験パートと LongMemEval

ハンズオンは `day2/03_context-engineering/Context_Engineering.ipynb` に集約されている。Chroma が公開した [Context Rot 研究](https://www.trychroma.com/research/context-rot) のミニ再現キットだと考えると分かりやすい。構成はこうだ。

1. **Part 1 — Repeated Words (faithfulness)**
   `apple` を N 回繰り返し、その中に 1 つだけ `apples` を混ぜたテキストを「そのまま再現せよ」と指示する。入力長を 25 → 50 → 100 → 250 → 500 → 1000 → 2500 語と振り、Levenshtein スコア・modified word の検出率・位置精度を測る。
2. **Part 2 — Needle In A Haystack (retrieval)**
   長文ドキュメント（"haystack"）の中に 1 文の "needle" を埋め込み、入力長 × 埋め込み深さ (depth) の grid で取り出し精度を測る。Chroma が公開している GPT-4.1 のヒートマップとも比較できる。
3. **Part 3 — 自分で実験設計**
   XML タグでの needle 囲み、extended thinking の有無、`<question>` ブロックの位置（document の前か後か）など、context 工学的介入が retrieval を改善するかをデザインして検証する。
4. **(Bonus) Part 4 — LongMemEval**
   約 113K token の会話履歴を「関連部分のみ (focused)」と「全部入り (full)」で渡したときの精度差を確認する。

ノートブックの冒頭にはこんな顧客シナリオが据えられている。

> "A customer's Claude-powered deal analysis agent scores **95% in staging** and **67% in production**. Their engineering lead says the model is probably the issue."

ステージングで 95% だった agent が本番で 67% に落ちる — 実務で何度も見る景色だ。「モデルを Opus に上げるべきか？」と相談されたとき、即答できる側に回るためのトレーニングだと思って臨んだ。

---

## 何を学んだか — context rot と "lost in the middle"

体感した気づきはいくつかある。

**Context rot は実在する**。入力長が伸びるほど、忠実性 (faithfulness) も検索性能 (retrieval) も劣化する。これは数値で確認した一次経験で、もう「たぶんそう」では戻れない。Part 1 のグラフを自分の手で出すと、Levenshtein スコアが 250 語あたりからほつれ始めるのが見える。

**"Lost in the middle" は最新モデルでも残る**。NIAH の grid を埋めていくと、中盤 25-50% depth の精度が一番低い谷になる。冒頭と末尾は強いが、真ん中に置いた重要情報はモデルの注意から滑り落ちやすい。

**XML マーカーは「無料の改善」になりうる**。Part 3 で `<key_information>...</key_information>` を 1 行足しただけで、長文の retrieval スコアが上がるケースを観測した。中身は変えていない、ただ境界をモデルに伝えただけだ。これは "魔法" ではなく、訓練データと RLHF の両方が XML 構造化に整合しているからだろう。

**focused は full に勝つ**。LongMemEval で 113K の履歴を丸ごと渡すより、関連部分だけに絞ったほうが、速くて、安くて、正確だった。「全部入れる」は楽だが、設計の放棄でもある。

これらを総合すると、Chroma が言う *context is a finite resource* というメッセージが体感として腑に落ちる。Anthropic の [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) は、この研究をプロダクション応用に翻訳したガイドであり、本章の実験はちょうどそのブリッジを実体験で渡る作業だった。

---

## Prompt と Context は別スキルだ

セッション中、講師がさらっと口にした整理が一番印象に残っている。

- **Prompt engineering**: 単一ターンの指示の最適化。XML、CoT、role 設定、few-shot など、テキストの "書き方" を磨く技術。
- **Context engineering**: エージェント向けに動的 context をキュレートし、ツールループの中で「何を残し / 落とし / 要約し / 外に逃がすか」を設計する技術。

Prompt の延長では context は扱えない、というのが認識を変えたポイントだった。前者は文章作法に近く、後者はソフトウェア設計に近い。履歴管理・ツール設計・メモリ管理を含む、より広い問題領域だ。

---

## エージェント向けの 3 プリミティブ

Anthropic が示すエージェント向けの context 管理プリミティブは 3 つ。これを覚えておくと、現場で「どこを直すか」の意思決定が速くなる。

### 1. Tool result clearing — 使い終わったツール出力を捨てる

巨大な検索結果や WebFetch の raw HTML を、後段の推論で必要なくなった時点で会話履歴から削除する。

```python
context_management = {
    "tool_result_clearing": {
        "trigger": {"type": "input_tokens", "value": 100_000},  # 100K で発火
        "keep": 3,                                              # 直近 3 件は残す
        "clear_at_least": 1,                                    # 最低 1 件は消す
        "exclude_tools": ["memory"],                            # memory ツールは保護
        "clear_tool_inputs": False,                             # 入力側は残す
    }
}
```

### 2. Compaction — 履歴が limit に近づいたら要約に置き換える

trigger threshold（何% で発火するか）と custom summary instructions（要約方針）を持つ。「ユーザの嗜好と未解決タスクを優先して保持せよ」のような自社ドメイン固有の要約方針が書ける。

### 3. Memory tool — 外部ストアへ退避する

ファイルバックの外部ストアに、モデル自身が「何を保存するか」を判断して書き出す。API は次の通り。

- `view` — 既存のメモリを参照
- `create` — 新しいエントリを作成
- `str_replace` — 部分置換
- `insert` — 既存エントリへ追記
- `delete` — エントリ削除

「ツール結果が大きい」のか「履歴が長くなる」のか「セッションを跨ぐ」のか、原因によって選ぶプリミティブが違う。万能の一手はない。

---

## 前提が崩れた瞬間、3 つ

このセッションで、自分の中で静かに崩れていった前提が 3 つある。

**(1) 1M context があるから全部入れて良い、と思っていた。**
入る ≠ 使われる、だった。Repeated Words 実験が示すのは、推論を要しない単なる複製ですら長文では崩れる、という事実。context window は「物理的に入る上限」であって「精度が出る上限」ではない。

**(2) 履歴は全部渡すほど賢くなる、と思っていた。**
LongMemEval は逆を突きつけてくる。focused のほうが full より明確に高精度で、しかも安くて速い。「全部入れる」は親切ではなく、設計を放棄しているだけだった。

**(3) context は「容量」だと思っていた。**
そうではなく、「資源」であり「設計対象」だった。何を残すか・捨てるか・要約するか・外に逃がすかを、ループのたびに決める。容量で語ると Opus に上げる話になるが、資源で語ると tool result clearing を入れる話になる。立っている地面が変わる感覚があった。

---

## 押さえておきたいコード／設定

技術的に手元に残しておきたいスニペットを 2 つだけ抜き出しておく。

### NIAH プロンプトの基本形

Part 2 の `create_niah_prompt` は、`<document_content>` と `<question>` を XML タグで囲む典型形。

```python
def create_niah_prompt(haystack_with_needle, question):
    return f"""You are a helpful AI bot that answers questions for a user. Keep your response short and direct

<document_content>
{haystack_with_needle}
</document_content>

Here is the user question:
<question>
{question}
</question>

Don't give information outside the document or repeat your findings.
Here is the most relevant information in the documents:"""
```

### XML タグで needle を囲む最小例

Part 3 の "worked example"。同じ情報でも、境界をモデルに明示するだけで attention が変わる。

```python
# Control: 平文の needle
control_haystack = insert_needle_at_depth(base, NEEDLE, TEST_DEPTH)
control_prompt = create_niah_prompt(control_haystack, QUESTION)

# Treatment: XML タグで囲んだ needle
tagged_needle = f"<key_information>{NEEDLE}</key_information>"
treatment_haystack = insert_needle_at_depth(base, tagged_needle, TEST_DEPTH)
treatment_prompt = create_niah_prompt(treatment_haystack, QUESTION)
```

---

## 現場に持ち帰りたい 7 か条

セッションを抜けて、メモ帳に書き残したのはこの 7 つだった。

1. **入れる量を減らす（最優先）。** 「念のため全部入れる」は最悪の選択。retrieval / pre-filter / summarization で関連 slice だけを渡す。
2. **重要情報は冒頭か末尾。** 中盤に埋めない。指示は質問の直前にも再掲する（sandwich pattern）。
3. **XML マーカーで重要箇所を明示する。** `<key_information>` `<document_content>` `<question>` の使い分けは、attention のチューニングコストとして安すぎる。
4. **Distractor は積極的に削る。** semantic distractor は短い context でも害があり、長い context では破壊的。top-k を増やすより減らす。
5. **3 プリミティブを使い分ける。** tool result clearing / compaction / memory tool は、何が context を圧迫しているかで選ぶ。
6. **推論を分割する。** 1 ショットで「長文 + 複雑な質問」を投げない。summarize-then-answer / chunk-and-retrieve / multi-turn に分解する。
7. **本番に近い形状で必ず eval を回す。** ベンチマーク 95% でも自社 context shape で 67% は普通に起こる。入力長分布・depth・distractor を反映した eval セットを持つ。

---

## もっと深掘りする入口

この章で扱ったのは Chroma 研究の再現と、その応用ガイドラインだ。もう一歩踏み込みたい人は、次の 2 本に直接当たるのが近道。

- Chroma: [*Context Rot: How Increasing Input Tokens Impacts LLM Performance*](https://www.trychroma.com/research/context-rot) — 本章のすべての実験設計の元ネタ。グラフを自分の目で確認したい人向け。
- Anthropic: [*Effective context engineering for AI agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 上の研究知見をエージェント設計にどう落とすか。3 プリミティブの位置づけも詳しい。

ほかに手元で何度も開く資料は次のあたり。

- Anthropic Docs - Long context tips: https://docs.anthropic.com/ja/docs/build-with-claude/prompt-engineering/long-context-tips
- Anthropic Docs - Use XML tags: https://docs.anthropic.com/ja/docs/build-with-claude/prompt-engineering/use-xml-tags
- LongMemEval (xiaowu0162/LongMemEval): https://github.com/xiaowu0162/LongMemEval

---

## クライマックスとしての回収

第1章で著者が掲げた takeaway #2 はこうだった。**「Context Engineering は nice-to-have ではなく、これが規律だ」**。最初に読んだときは、正直、標語くらいに受け取っていた。

この章を実験で回ってみて、その標語が一気に重みを持った。凡庸な AI システムと、本番でも信頼できる AI システムの差は、多くの場合「どのモデルを使ったか」ではなく「どの情報を、どの形で、どの位置で、いつ与えたか」で決まる。モデル選定や temperature の調整より手前に、context の設計という規律がある。staging 95% / production 67% という落差の大半は、ここで説明できてしまう。

> "LLM はコンテキストを忠実に読む保証はない。だから、読ませたいものだけを、読ませたい順に、読ませたい形で渡す。"

このフレーズを、本書のクライマックスとして置いておきたい。Context は容量ではなく資源であり、設計対象であり、規律である — それを目撃した 2 日目の午後の話。

次章 [09-agent-build-hackathon](./09-agent-build-hackathon) では、ここまで学んだ Prompt / Eval / Inference / Context のすべてを束ねて、半日で一本のエージェントを立てるハッカソンに突入する。
