---
title: "Day2-03: Context Engineering — Context Rot との戦い"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day2/03_context-engineering/`
> **参考研究**:
> - Chroma's *Context Rot: How Increasing Input Tokens Impacts LLM Performance*: https://www.trychroma.com/research/context-rot
> - Anthropic *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

---

## はじめに — 膨大な入力に強いという思い込みが折れた

2 日目の午後、Context Engineering のセッションに、自分はわりと余裕で入った。Prompt の書き方ならそこそこ自信があったし、1M token の context window を持つモデルも触ったことがあった。「膨大な入力にも耐えられるようになったんでしょ、知ってる」くらいの心持ちで椅子に座っていた。

その心持ちは、最初の実験パートで完全に折れた。`apple` をひたすら並べた中に 1 つだけ `apples` を混ぜ、「そのまま再現せよ」とだけ命じる、推論を一切要しない単純な複製タスク。語数を増やしていくと、ある地点からモデルは普通に「直して」しまう。文字列を写すだけのタスクですら、入力が長くなれば崩れていく — その挙動を目の前のグラフで見た瞬間、「とりあえず context に全部入れておけば良い」という自分の悪癖が、この研修の中で露呈された。

本章では、その実験で測れたこと・そこから抽出した原則・現場で守りたいアンチパターン回避を、まずファクトとして整理する。個人的に何が響いたのかは、章の後半に記載する。

---

## 題材 — 3 つの実験パートと LongMemEval

ハンズオンは `day2/03_context-engineering/Context_Engineering.ipynb` に集約されている。Chroma が公開した [Context Rot 研究](https://www.trychroma.com/research/context-rot) のミニ再現キットだと考えると分かりやすい。冒頭には次の顧客シナリオが据えられている。

> "A customer's Claude-powered deal analysis agent scores **95% in staging** and **67% in production**. Their engineering lead says the model is probably the issue."

ステージングで 95% だったエージェントが本番で 67% に落ちる。エンジニアリングリードは「モデルが原因だろう」と言っているが、本当にそうか — その診断ツールキットとして 4 パートが組まれている。

1. **Part 1 — Repeated Words (faithfulness)**
   `apple` を N 回繰り返し、中に 1 つだけ `apples` を混ぜたテキストを「そのまま再現せよ」と指示する。入力長を 25 → 50 → 100 → 250 → 500 → 1000 → 2500 語と振り、Levenshtein スコア[^levenshtein]・modified word の検出率・位置精度を測る。推論を一切要しない、純粋な「忠実性」のテスト。
2. **Part 2 — Needle In A Haystack (retrieval)**
   長文 (`<document_content>`) の中に 1 文の "needle"（特定の情報）を埋め込み、入力長 × 埋め込み深さ (depth) のグリッドで取り出し精度を測る。Chroma が公開している GPT-4.1 のヒートマップとも比較できる。
3. **Part 3 — 自分で実験設計**
   XML タグでの "needle" 囲み、extended thinking の有無、`<question>` ブロックの位置（document の前か後か）など、context 工学的介入が retrieval を改善するかを設計して検証する。
4. **(Bonus) Part 4 — LongMemEval**
   約 113K token の会話履歴を「関連部分のみ (focused)」と「全部入り (full)」で渡したときの精度差を確認する。ベースラインとして GPT-4.1 の結果が CSV で同梱されている。

中心命題は最初から最後までブレない。1M context window は capacity であって設計ターゲットではない。つまり、「何トークンまで入るか」は重要ではなく、**「何を入れて何を捨てるか」を設計** すべきということだ。staging 95% → production 67% という落差の大半は、モデル選定ではなく context 設計問題で説明できてしまう、という仮説を、自分の手で検証しに行く。

---

## ベストプラクティス・アンチパターン・重要ポイント

ここからは、ハンズオンと Chroma / Anthropic の研究から抽出した原則を列挙する。各項は「原則 → アンチパターン → 具体例」の順で書く。

### Context は有限資源として設計する

:::message
**原則**: Context は容量ではなく資源である。1M token 入るという物理上限は「精度が出る上限」を意味しない。何を残し、何を捨て、何を要約し、何を外部に逃がすか — エージェントループのターンごとに決める設計対象である。
:::

:::message alert
**アンチパターン**: 「念のため全部入れる」。retrieval も pre-filter も summarization も挟まず、生のドキュメントと履歴を丸ごと毎ターン同梱する設計。短期的には楽だが、context rot[^context-rot] の影響を最も強く受ける。
:::

#### **ハンズオンでの具体例**

LongMemEval で 113K token の会話履歴を full で渡したケースと、関連ターンに絞った focused で渡したケースを比較すると、focused のほうが速く・安く・正確だった。「全部渡せば賢くなる」ではなく、「絞ったほうが賢くなる」が観測された一次データになる。

-----

### Context rot は curve であって閾値ではない

:::message
**原則**: 入力長が伸びるほど、忠実性 (faithfulness) と検索性能 (retrieval) は連続的に劣化する。「N トークンまでは安全、それ以降は危険」という閾値ではなく、長くなるほど少しずつ崩れていくカーブとして理解する。
:::

:::message alert
**アンチパターン**: 「context window 上限まではフルに使ってよい」と前提を置くこと。劣化は容量を使い切る手前から始まる。
:::

#### **ハンズオンでの具体例**

Part 1 の Repeated Words は、推論を一切要しないただの複製タスクであるにもかかわらず、語数を 25 → 50 → 100 → 250 → 500 → 1000 → 2500 と振っていくと Levenshtein スコアが緩やかに落ちる。`apple` x500 個の中に 1 個だけ `apples` を混ぜて「そのまま写して」と命じても、長くなるにつれモデルは普通に `apple` に「直して」しまう。

-----

### "Lost in the middle" は位置で精度が揺らぐ

:::message
**原則**: 同じ情報でも、document の冒頭・末尾に置くか、中盤に置くかで取り出し精度が変わる。最弱は **25–50% depth** の中盤帯。重要情報は冒頭か末尾に置く。指示は質問の直前にも再掲する（sandwich pattern）。
:::

:::message alert
**アンチパターン**: 長文の中盤に重要情報を埋めて、モデルが拾ってくれることを期待する設計。Chroma の grid を見れば、中盤が一番暗い帯になることが先に分かっている。
:::

#### **ハンズオンでの具体例**

Part 2 の NIAH ヒートマップで `needle_depth=25` `needle_depth=50` の行が、冒頭 (0%) や末尾 (100%) の行より明確に低い accuracy を示す。Anthropic Docs の [Long context tips](https://docs.anthropic.com/ja/docs/build-with-claude/prompt-engineering/long-context-tips) も「重要情報は末尾近くに置け」を明示している。

-----

### Prompt Engineering と Context Engineering は別スキル

:::message
**原則**: Prompt Engineering は単発ターンの言葉遣い（XML、CoT、role 設定、few-shot 等）の最適化、Context Engineering はエージェントループの中で「何を残し / 落とし / 要約し / 外に逃がすか」を多ターンにわたって設計する技術。問題領域も使う道具も別物である。
:::

:::message alert
**アンチパターン**: 「prompt を長く詳しく書く」ことを context engineering と呼ぶ。長い prompt は単に長い prompt であり、context rot を悪化させる側に効く。
:::

#### **ハンズオンでの具体例**

Prompt Engineering の延長で「指示と参照情報を全部 system prompt に詰める」と、毎ターンその全量がコンテキストを食い続ける。Context Engineering 視点では、参照情報は memory tool に逃がすか、retrieval で必要分のみ持ってくる構造に切り替える。

-----

### 3 primitives は問題に応じて選ぶ

:::message
**原則**: Anthropic が示すエージェント向け context 管理プリミティブは 3 つ。何が context を圧迫しているかによって選ぶ。
:::

| 圧迫源                       | 基本要素      | 役割                                       |
| ---------------------------- | --------------------- | ------------------------------------------ |
| ツール出力が大きい           | Tool result clearing  | 使い終わった結果を会話履歴から落とす       |
| 履歴自体が長い               | Compaction            | 履歴を要約に置き換える                     |
| セッションを跨いで保持したい | Memory tool           | 外部ストアに退避し、必要時に取り出す       |

:::message alert
**アンチパターン**: 原因を見ずに「とりあえず compaction を入れる」「とりあえず memory tool を入れる」。tool 出力が暴れているのに compaction を入れても、要約結果がさらに暴れる。
:::

#### **ハンズオンでの具体例**

WebFetch や検索系ツールの raw HTML/JSON が会話履歴を肥大化させているケースは `Tool result clearing` が第一手。複数ターンにわたる雑談・前提共有で履歴が伸びているケースは `Compaction`。プロジェクトを跨いだユーザ嗜好の記憶などは `Memory tool`。

-----

### Tool Result Clearing の設定項目

:::message
**原則**: Tool Result Clearing は次の 5 つの設定で挙動を制御する。
:::

- `trigger` — 何で発火するか（例: `input_tokens` が 100K 超）
- `keep` — 直近何件のツール結果を残すか
- `clear_at_least` — 1 回の発火で最低何件落とすか
- `exclude_tools` — 保護したいツール名のリスト（memory 等）
- `clear_tool_inputs` — 入力側も落とすかどうか

:::message alert
**アンチパターン**: `exclude_tools` を指定せずに memory tool 等の「落としてはいけないツール出力」まで巻き込んで削除する。落とすべきは raw な検索結果や fetch 結果（WebFetchやログ出力など）であって、永続化責務を持つツールの返り値ではない。
:::

#### **ハンズオンでの具体例**


```python
context_management = {
    "tool_result_clearing": {
        "trigger": {"type": "input_tokens", "value": 100_000},
        "keep": 3,
        "clear_at_least": 1,
        "exclude_tools": ["memory"],
        "clear_tool_inputs": False,
    }
}
```

-----

### Memory tool は外部ストレージ責務

:::message
**原則**: Memory tool はファイルバックの外部ストアに対する CRUD インターフェースであり、モデル自身が「何を保存するか / 更新するか / 消すか」をループ中に決める。API は `view` / `create` / `str_replace` / `insert` / `delete`。
:::

:::message alert
**アンチパターン**: 会話履歴を全部 memory に書き込む。memory はあくまで「セッションを跨いで残す必要があるもの」用であり、短期記憶の代替ではない。短期記憶は compaction が担う。
:::

#### **ハンズオンでの具体例**

ユーザの恒久的な嗜好（「敬語を使わない」「コードブロックには言語タグを付ける」）や、複数日にまたがるプロジェクト固有の前提を `create` / `str_replace` で更新する。1 ターン限定の検索結果は memory に書かない。

-----

### XML タグは注意機構として使う

:::message
**原則**: `<key_information>` `<document_content>` `<question>` のような XML タグで境界をモデルに明示すると、retrieval スコアが改善するケースがある。中身を変えていなくても、注意機構（attention）が乗りやすくなる。Anthropic の訓練データと RLHF が XML 構造化に整合しているためと考えられる。
:::

:::message alert
**アンチパターン**: 全段落に `<important>` を付ける。「全部重要」は「どれも重要ではない」と意味的に等価で、注意機構を狂わせる方向に効く。
:::

#### **ハンズオンでの具体例**

Part 3 の "worked example" では、同一の needle を `<key_information>...</key_information>` で囲んだ treatment と、平文の control で NIAH 精度を比較する。境界を明示しただけで attention が変わる。詳細は Anthropic Docs の [Use XML tags](https://docs.anthropic.com/ja/docs/build-with-claude/prompt-engineering/use-xml-tags) を参照。

-----

### 「全部突っ込む」は怠惰、focused が勝つ

:::message
**原則**: Distractor（意味的に似ているが無関係な情報）は short context でも害があり、long context では破壊的に効いてしまう。top-k を増やすより減らす。focused 設計のほうが、速さ・コスト・精度のすべてで勝つ。
:::

:::message alert
**アンチパターン**: 「retrieval の取りこぼしを防ぐため」と top-k を機械的に増やす。distractor の混入確率が上がり、retrieval は逆に劣化する。
:::

#### **ハンズオンでの具体例**

LongMemEval の focused vs full で、focused のほうが accuracy が高い。113K 全部を読ませることは「親切」ではなく「設計の放棄」である。

-----

### 本番形状で eval する

:::message
**原則**: 標準ベンチマークでの 95% は、自社の context shape での 95% を保証しない。入力長分布・needle depth・distractor 構成を、本番に近い形で揃えた eval セットを用意し、context engineering の変更ごとに回す。
:::

:::message alert
**アンチパターン**: staging 環境で短い文書だけで eval し、失敗要因が本番の長文 + 履歴 + tool 出力など混在する状態で初めて性能を測る。本章冒頭の staging 95% / production 67% は、まさにこれが原因の典型。
:::

#### **ハンズオンでの具体例**

Part 2 で組んだ `INPUT_LENGTHS × DEPTHS` のグリッドそのままを、自社データの入力長分布と needle 配置に合わせて再構成すれば、最小限のプロダクション eval ハーネスになる。

---

## 押さえておきたいコード/設定

実装側で手元に残しておきたいスニペットを抜き出しておく。

### NIAH プロンプトテンプレート

`<document_content>` と `<question>` を XML タグで囲み、最後に "Here is the most relevant information in the documents:" でモデルの出力を導出する基本形。

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

### 指定 depth に needle を差し込む関数

文字数ベースで挿入位置を求め、文末の区切り（ピリオド + 半角スペース）に揃えて挿入する。これにより文の途中に needle が割り込むのを避ける。

```python
def insert_needle_at_depth(haystack, needle, depth_percent):
    if depth_percent >= 100:
        return haystack + " " + needle
    if depth_percent <= 0:
        return needle + " " + haystack

    point = int(len(haystack) * (depth_percent / 100))
    boundary = haystack.rfind(". ", 0, point)
    if boundary > 0:
        point = boundary + 2
    return haystack[:point] + needle + " " + haystack[point:]
```

### XML タグ介入の最小比較

同じ needle・同じ haystack・同じ depth で、平文 vs `<key_information>` 付きを比較する。差分は 1 行のラップだけ。

```python
# Control: 平文の needle
control_haystack = insert_needle_at_depth(base, NEEDLE, TEST_DEPTH)
control_prompt = create_niah_prompt(control_haystack, QUESTION)

# Treatment: XML タグで囲んだ needle
tagged_needle = f"<key_information>{NEEDLE}</key_information>"
treatment_haystack = insert_needle_at_depth(base, tagged_needle, TEST_DEPTH)
treatment_prompt = create_niah_prompt(treatment_haystack, QUESTION)
```

### Tool Result Clearing の設定例

`trigger` / `keep` / `clear_at_least` / `exclude_tools` / `clear_tool_inputs` の 5 項目を Anthropic SDK 側で渡す形。

```python
context_management = {
    "tool_result_clearing": {
        "trigger": {"type": "input_tokens", "value": 100_000},
        "keep": 3,
        "clear_at_least": 1,
        "exclude_tools": ["memory"],
        "clear_tool_inputs": False,
    }
}
```

---

## よくある勘違いと気づき

- 勘違い：1M context があるから全部入れて良い
  > 入る ≠ 使われる、だった。`Part1: Repeated Words` 実験が示すのは、推論を要しない単なる複製ですら長文では崩れる、という事実。context window は「物理的に入る上限」であって「精度が出る上限」ではない。`apple` を 500 個並べた中の 1 個の `apples` を、モデルは「直して」しまう。

- 勘違い：履歴は全部渡すほど賢くなる
  > LongMemEval は逆を突きつけてくる。focused のほうが full より明確に高精度で、しかも安くて速い。「全部入れる」は親切ではなく、設計を放棄しているだけだった。人間で言えば「目に入る」（コンテキストウィンドウが大きい）と「認知できる」（重要な情報にフォーカスできる）が別物であるのと同じだ。

- 勘違い：context は「容量」である
  > contextは「資源」であり「設計対象」である。何を残すか・捨てるか・要約するか・外に逃がすかを、ループのたびに決める。容量で語ると `Opus` に上げる話になるが、資源で語ると `tool result clearing` を入れる話になる。

- 勘違い：Prompt engineering の延長で Context engineering を語れる
  >「Prompt engineering は単一ターンの指示の最適化、Context engineering はエージェント向けに動的 context をキュレートする技術」という整理が、この理解を一番きれいに言語化していた。前者は文章作法に近く、後者はソフトウェア設計に近い — その粒度の違いを、Repeated Words のグラフが視覚的に教えてくれた。

---

## 現場で実践したいこと

- **入れる量を減らす（最優先）**
  - 「念のため全部入れる」は最悪の選択。`retrieval` / `pre-filter` / `summarization` で関連 slice だけを渡す。

- **重要情報は冒頭か末尾**
  - 中盤に埋めない。指示は質問の直前にも再掲する（sandwich pattern）。

- **XML マーカーで重要箇所を明示する**
  - `<key_information>` `<document_content>` `<question>` の使い分けは、attention のチューニングコストとしてコスパが良い。ただし全部にタグを付けすぎない。

- **Distractor は積極的に削る**
  - semantic distractor は短い context でも害があり、長い context では破壊的。top-k を増やすより減らす。

- **3 プリミティブを使い分ける**
  - `tool result clearing` / `compaction` / `memory tool` は、何が context を圧迫しているかで選ぶ。原因と治療の対応を間違えない。

- **推論を分割する**
  - 1 ショットで「長文 + 複雑な質問」を投げない。summarize-then-answer / chunk-and-retrieve / multi-turn に分解する。

- **本番に近い形状で必ず eval を回す**
  - ベンチマーク 95% でも自社 context shape で 67% は普通に起こる。入力長分布・depth・distractor を反映した eval セットを持つ。

---

## さらなる理解のために

この章で扱ったのは Chroma 研究の再現と、その応用ガイドラインだ。もう一歩踏み込みたい場合は、次の 2 本に直接当たるのが近道。

- Chroma: [*Context Rot: How Increasing Input Tokens Impacts LLM Performance*](https://www.trychroma.com/research/context-rot) — 本章のすべての実験設計の元ネタ。グラフを自分の目で確認したい人向け。
- Anthropic: [*Effective context engineering for AI agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 上の研究知見をエージェント設計にどう落とすか。3 プリミティブの位置づけも詳しい。

ほかに手元で何度も開く資料は次のあたり。

- Anthropic Docs - Long context tips: https://docs.anthropic.com/ja/docs/build-with-claude/prompt-engineering/long-context-tips
- Anthropic Docs - Use XML tags: https://docs.anthropic.com/ja/docs/build-with-claude/prompt-engineering/use-xml-tags
- LongMemEval (xiaowu0162/LongMemEval): https://github.com/xiaowu0162/LongMemEval

---

## 章末 — クライマックスとしての回収

第1章で著者が掲げた takeaway #2 はこうだった。**「Context Engineering は nice-to-have ではなく、これが規律だ」**。最初に読んだときは、正直、標語くらいに受け取っていた。

この章で実験してみて、その標語が一気に重みを持った。凡庸な AI システムと、本番でも信頼できる AI システムの差は、多くの場合「どのモデルを使ったか」ではなく「どの情報を、どの形で、どの位置で、いつ与えたか」で決まる。モデル選定や temperature の調整より手前に、context の設計という規律がある。staging 95% / production 67% という落差の大半は、ここで説明できてしまう。

> "LLM はコンテキストを忠実に読む保証はない。だから、読ませたいものだけを、読ませたい順に、読ませたい形で渡す。"

このフレーズを、本書のエピローグとして添える。Context は容量ではなく資源であり、設計対象であり、規律である — それを目撃した 2 日目の午後の話。

→ 次章: [09-agent-build-hackathon](./09-agent-build-hackathon.md) では、ここまで学んだ Prompt / Eval / Inference / Context のすべてを束ねて、半日で一本のエージェントを立てるハッカソンに突入する。

[^levenshtein]: 2つの文字列の「編集距離」を数値化した指標。一方の文字列をもう一方に変換するのに必要な挿入・削除・置換の最小回数で表す。スコアが低いほど元のテキストに忠実に再現できていることを意味する。
[^context-rot]: 入力トークン数が増えるほど LLM の出力品質（忠実性・検索精度）が連続的に劣化していく現象。「N トークンまでは安全」という閾値は存在せず、長くなるにつれて少しずつ崩れていくカーブとして発生する。Chroma が公開した研究 [*Context Rot: How Increasing Input Tokens Impacts LLM Performance*](https://www.trychroma.com/research/context-rot) で命名・定量化された。
