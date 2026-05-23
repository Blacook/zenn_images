---
title: "Developer Platform — 5つの足場と Messages API の中身"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day1/02_developer-platform/`
> **題材**: TechFlow（中堅 B2B SaaS、500+ tickets/day）のサポートチケット triage エージェント
> **モデル**: `claude-sonnet-4-6`

## はじめに — フレームワークを外したら、ループが見えた

サンフランシスコの2日目、`while response.stop_reason == "tool_use":` という一行を自分の指で書いた瞬間のことを、いまでもよく覚えている。

それまで、エージェントは私にとって「LangChain や Agent SDK が裏でうまくやってくれる何か」だった。`agent.run()` を呼ぶと、ツールが順番に叩かれ、最後にきれいな結果が返ってくる。便利だけれど、何かが起きているはずなのに、その何かの輪郭がいつもぼやけていた。

現地のハンズオンで Messages API を直接叩き、TechFlow のサポートチケット triage エージェントを **何のフレームワークも噛まさずに** 組み立てたとき、その霧が一気に晴れた。エージェントとは、要するに **`stop_reason` を見ながら回す while ループ** だった。ツール呼び出しは、JSON で帰ってきた `tool_use` ブロックを実装にディスパッチし、結果を `tool_result` で次のターンに積んでいるだけだった。隠されていたのは魔法ではなく、ただの制御構造だったのだ、と腹落ちした。

この章はまず「エージェントに知識とツールアクセスを与える足場は何種類あり、いつどれを選ぶか」という地図を広げる。そのうえで、地図の真ん中にある Messages API に潜り、tool use・adaptive thinking・structured output・streaming の 5 要素がどう合成されてひとつのエージェントになるのかを、自分の言葉で振り返る。

### 講義の入口 — 「12 個の MCP を建てるべきか？」

セッションの冒頭で、講師は会場に対してこう問いを置いた。「自社のデータ源それぞれに対して、専用の MCP サーバを 12 個立てるべきか？」 ── 多くの参加者が直感的に頷きかける問いだった。だが講師の答えは即座に「No」だった。

「あなたが持っているのは MCP だけじゃない。CLI access、API、Skills、そして Plugins という別の足場がある。それぞれの得意分野を素直に使い分ければよくて、すべてを MCP で揃える必要はない。」 ── これがこの章の出発点になる。以下、5つの足場を順に並べ、最後にもう一度この問いに戻る。

## 題材 — TechFlow の Tier 1 を肩代わりする

題材は中堅 B2B SaaS の TechFlow。Tier 1 サポートが 1 日 500 件超のチケットを 1 件あたり約 8 分で捌いている、という設定だ。これを Claude に置き換える。

エージェントが持つ道具は 3 つだけだった。

| ツール | 役割 |
|---|---|
| `get_ticket` | チケット ID から本文・顧客・優先度・プロダクト領域を取得 |
| `search_kb` | キーワードで Knowledge Base を検索 |
| `resolve_ticket` | 解決ノートを書いてチケットを `resolved` / `escalated` / `closed` に遷移 |

道具立てが小さいぶん、エージェントの「外側」をどう組むかに意識を集中できる。これがハンズオン題材としてとても良かった。

## 何を学んだか — 5つの足場と Messages API の中身

### 5つの足場 — CLI / API / MCP / Skills / Plugins

エージェントに「外の世界」を触らせる手段は今や少なくない。決定的な正解は無いが、講師は5つの足場をそれぞれの **向き不向き／壊れ方／いつ選ぶか** で並べてくれた。

### CLI access — POC の最短距離

- **向き不向き**: 足場が極小、Bloomberg / Shopify / Snowflake のような既存データ源を「とにかく繋いで、モデルに食わせる」のが最速でできる。「これだけのデータを渡せばこれだけ出るんですよ」を見せるデモ・POC に最適。
- **壊れ方**: 長期運用に耐えない。スケールしない。
- **いつ選ぶか**: POC・デモ・探索フェーズ限定。クライアントへの初回提案で「もし内部データを渡したらこうなる」を示す瞬間に限る。

### API — 予測可能・繰り返しに強い

- **向き不向き**: エコシステムにある既存ソフトはすでに API を叩いている。「毎回同じ入力で同じ出力が返ってくる」種類のやり取りには、これ以上ない最適解。
- **壊れ方**: エージェントが「毎回違う 4 つのシステムに違う方法でアクセスする」種類の柔軟性を要求しはじめると破綻する。固定スキーマに引きずられて、エージェントが取りたい挙動を取れなくなる。
- **いつ選ぶか**: 入出力が安定し、繰り返し性が高いとき。エージェントの判断ではなく仕様で挙動が決まる部分を担う。

### MCP — M×N を M+N に畳む

- **向き不向き**: 旧世界では M クライアント × N サービスで M×N の統合点を抱えていたが、MCP はクライアントとサーバを分離することで統合点を M+N に減らす。クライアント側は同じインターフェイスのままサーバ側を差し替えられるので、上層のコードを書き直さずにツールを増減できる。LLM はプロトコルを「箱から出してすぐ」理解できるので、エージェントとの相性も良い。
- **壊れ方**: ツールを繋ぎすぎると context bloat が起きる。**目安として 15 ツールを超えたあたり** から、モデルは人間の従業員と同じく「どのツールをいつ使うか」を取り違えはじめる。
- **いつ選ぶか**: 入力が不完全でも問題を解く柔軟性が必要なとき、複数システムを跨いだ orchestration が必要なとき。講師が挙げた小ネタが象徴的だった ── 「4 つのシステムを横断するエージェントが、途中で Salesforce のあるデータを取りこぼしているのに、それに気づかず最後まで進んでしまう」。この **取りこぼしを catch & fix できる場所** が MCP のオーケストレーション層だ、というのが MCP の価値の核心らしい。

### Skills — 知識層としての足場

会場でも一番質問が集まったのがこの Skills だった。講師の語り口を整理すると、Skills が機能するための骨格は4つに集約される。

- **Progressive disclosure** ── Claude はまず title と description だけを読んで「このスキルに踏み込むか」を判断する。本文はそのスキルが必要だと判断したあとで初めて context に積まれる。これが context budget を節約する核になっており、Skills という形式の設計思想の中心にある。
- **Title と description を、目的特化で短く書く** ── 例として「front-end design」スキルが紹介された。description は超タイト、name も "front end design" のように直接的。曖昧さを早い段階で潰すことで、無駄なロードが起きない。
- **Directional commitment** ── 「modern」のような主観的な形容詞を排除し、typography / motion / spatial composition のように **方向を確定する具体性** を入れる。「最も賢い存在として振る舞え」のような中身のない強調はモデルを賢くしない、というのが講師の繰り返した警句だった。
- **Hard negatives** ── 「絶対にこうするな」のリストをスキル本文に書く。エンタープライズで Skills を運用している組織は、本番で観測した失敗パターンを 5〜6 回観測したら必ずスキルに反映する、というワークフローを持っている。出力品質の改善余地が一番大きいのは、実はここだという。

### Plugins — 合成レイヤ

- 上の足場（Skills・MCP・API）をまとめてパッケージにするためのレイヤ。Anthropic 公式で finance / data analyst のような Plugin が公開されはじめており、Claude Code UI 内に marketplace 的な入口がある。
- 個別の Skill や MCP を毎回手で組み合わせるのではなく、「ユースケース単位のひと固まり」として配布できるので、配布性と再現性が高まる。

### 「結局どう選ぶか」— 講師が最後に出した答え

冒頭の問いに戻る。

> 「12 個の MCP を建てるべきか？」 ── No。
> API に得意なこと（予測可能・繰り返し）をやらせ、MCP に得意なこと（柔軟な orchestration）をやらせ、その上に Skills という知識層を被せて、モデルが「いつ何を呼ぶか」を導けるようにする。Plugins はそれらをユースケース単位に束ねて配布する入れ物。

これは現地で繰り返し提示された **層の構成図** で、私が Bootcamp で受け取った最も実務的なアーキテクチャ観のひとつになった。

### 5 要素がどう合成されるか — Messages API の積み方

ハンズオンノートブック `Developer_Platform.ipynb` は、同じ triage タスクの上に要素を 1 枚ずつ重ねていく構成になっていた。順序自体が学びだったので、その並びで振り返る。

最初に **ツール schema** を書く。`name` と `description`、`input_schema` の 3 点セットだ。書きながら気づいたのは、`description` は Claude にとっての **API ドキュメント** だということ。曖昧に書けば曖昧に呼ばれるし、「いつ呼ぶか」「いつ呼ばないか」まで書けば、ツール選択ミスが目に見えて減る。

次に **agentic loop**。`response.stop_reason == "tool_use"` のあいだ、`tool_use` ブロックを実装にディスパッチし、結果を `tool_result` として messages に積み、もう一度 `messages.create()` を叩く。それだけ。`end_turn` が返ってきたらループを抜ける。これが本当に、本当に、それだけだった、というのが一番の発見だった。

そこに **structured output** を被せる。ループが終わったあと、もう 1 度だけ Claude を呼んで、解決サマリーを JSON Schema に従って取り出す。ここで `output_config.format` を初めて指定する。

さらに **adaptive thinking + `effort`** を全コールに足す。`thinking={"type": "adaptive"}` で「複雑なら深く、簡単なら浅く」を任せ、`effort` で深さの最大幅を握る。

最後に **streaming**。`client.messages.stream()` に切り替えると、`content_block_start` → `content_block_delta` → `content_block_stop` のイベントが時系列で流れてくる。`thinking_delta` と `text_delta` と `input_json_delta` を別々にハンドリングすると、思考・応答・ツール引数が **それぞれ独立した流れとして見える**。これも、SDK 越しでは絶対に体感できなかった景色だった。

これら 5 要素は、互いに無関係に積めるわけではなく、**合成順序とスコープに依存関係がある**。それを身体で覚えるのが、このハンズオンの本当の目的だったように思う。

## 前提が崩れた瞬間

「フレームワーク無しで書く」というのは、つまり **誤解していた前提が次々と剥がれる** ということでもあった。記憶に残っているものをそのまま並べる。

**思考ブロックは next call に渡さなくていい、と思っていた。** これは違った。`response.content` に `ThinkingBlock` が混ざっていたら、`text` や `tool_use` と一緒に **そのまま** 次の `messages` に積まないといけない。落とすと整合性エラーで弾かれる。extended thinking はサーバ側で連続性を検証している、という説明を聞いて、「ああ、状態を持っているのか」と理解した。

```python
# 私が最初にやってしまった書き方（壊れる）
filtered = [b for b in response.content if b.type != "thinking"]
messages.append({"role": "assistant", "content": filtered})

# 正解
messages.append({"role": "assistant", "content": response.content})
```

**`effort="high"` にしておけば安心、と思っていた。** これも違った。`high` は確かに深く考えてくれる。でもレスポンス時間が数倍に伸び、思考トークンも全部課金されるので、500 件/日のスケールでは経済合理性が一瞬で崩れる。請求の二重課金のような単純なチケットは `low` で十分捌けるし、API 500 エラーの間欠障害のような曖昧なチケットでのみ `high` に振る、というルーティングが現実解だ、というのが講師の言だった。同じチケットを `low` と `high` で並べて投げる演習があり、`high` のほうの思考トレースに "Singapore region routing issue" みたいな仮説が立っているのを見て、「これは確かに `high` が要る」と納得した。

**`output_config.format` は最初から付けっぱなしでよさそう、と思っていた。** これは一番派手にハマるパターンらしい。`format` を指定すると、Claude の **すべてのテキスト出力が JSON Schema に制約される**。ツールループ中にこれを有効にしていると、Claude が `tool_use` を返したい場面で「テキストは JSON でないとダメ」と引っ張られ、ツール呼び出しが壊れる。**ループ中は `effort` だけ、最終コールでだけ `format`**、というのが鉄則だった。

```python
# ループ中
output_config={"effort": effort}

# 最終構造化出力
output_config={"effort": effort, "format": RESOLUTION_SCHEMA}
tool_choice={"type": "none"}  # ここも忘れずに
```

**`response.content` の最初の text ブロックを読めばよい、と思っていた。** これも罠だった。adaptive thinking 有効時、`content` は `[ThinkingBlock, TextBlock, ...]` の並びで返ってくる。JSON は **最後の** text ブロックに入っている。最初を見にいくと、`thinking` の手前にある中間的なテキストを掴んでしまうことがある。

```python
text_blocks = [b for b in response.content if b.type == "text" and b.text.strip()]
data = json.loads(text_blocks[-1].text)
```

どれも、SDK が抽象化してくれていたら一生気づかなかったたぐいの肌触りだ。

## 押さえておきたいコード／設定

ノートブックの該当部分を、自分が後で見返したくなる粒度に絞って残しておく。完全版は `day1/02_developer-platform/Developer_Platform.ipynb` にある。

### クライアント初期化

API キーは環境変数経由で。**ノートブックや Git にキーを書かない**、というのは現地でも何度も念押しされた。

```python
import os
import anthropic

client = anthropic.Anthropic(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    timeout=900.0,  # max_tokens を大きく取る場合に必要
)
MODEL = "claude-sonnet-4-6"
```

### ツール schema の最小例

```python
tools = [
    {
        "name": "search_kb",
        "description": (
            "Search the knowledge base for articles relevant to a customer issue. "
            "Use this to find troubleshooting steps, policies, or known solutions "
            "before attempting to resolve a ticket."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": (
                        "A free-text keyword query describing the issue, e.g. "
                        "'webhook auth error after key rotation'"
                    ),
                }
            },
            "required": ["query"],
        },
    },
    # get_ticket / resolve_ticket も同じ調子で
]
```

`description` の中に "before attempting to resolve a ticket" のような **順序のヒント** を書くと、Claude のツール選択が安定する、というのは現地で何度も目撃した。

### Agentic loop の骨格

```python
def run_agent(user_message: str):
    messages = [{"role": "user", "content": user_message}]

    response = client.messages.create(
        model=MODEL,
        max_tokens=32000,
        system=SYSTEM_PROMPT,
        tools=tools,
        thinking={"type": "adaptive"},
        messages=messages,
    )

    while response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result),
                })

        # thinking ブロックも含めて、content をそのまま積む
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

        response = client.messages.create(
            model=MODEL,
            max_tokens=32000,
            system=SYSTEM_PROMPT,
            tools=tools,
            thinking={"type": "adaptive"},
            messages=messages,
        )

    return response
```

このコードを自分の手で打ち、`end_turn` がちゃんと返ってきてループが抜ける瞬間を見たとき、本当にエージェントは「ただの while」なんだ、と笑ってしまった。

### 最終構造化出力の差分

```python
RESOLUTION_SCHEMA = {
    "type": "json_schema",
    "schema": {
        "type": "object",
        "properties": {
            "diagnosis": {"type": "string"},
            "solution_steps": {"type": "array", "items": {"type": "string"}},
            "confidence": {"type": "string", "enum": ["high", "medium", "low"]},
            "escalation_needed": {"type": "boolean"},
            "category": {
                "type": "string",
                "enum": ["billing", "technical", "account", "feature_request"],
            },
        },
        "required": [
            "diagnosis", "solution_steps", "confidence",
            "escalation_needed", "category",
        ],
        "additionalProperties": False,
    },
}

final = client.messages.create(
    model=MODEL,
    max_tokens=8000,
    system=SYSTEM_PROMPT,
    output_config={"effort": effort, "format": RESOLUTION_SCHEMA},
    tool_choice={"type": "none"},
    thinking={"type": "adaptive"},
    messages=messages,
)
```

`additionalProperties: False` は地味だが重要だ、と感じた。これを書かないと Claude が schema にないフィールドを足してくることがあり、下流のパースが死ぬ。

### Streaming のイベント分岐

```python
with client.messages.stream(
    model=MODEL,
    max_tokens=32000,
    system=SYSTEM_PROMPT,
    tools=tools,
    thinking={"type": "adaptive"},
    output_config={"effort": effort},  # ループ中は format を入れない
    messages=messages,
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            block = event.content_block
            if block.type == "thinking":
                print("\n[Thinking] ", end="", flush=True)
            elif block.type == "tool_use":
                print(f"\n[Tool: {block.name}] ", end="", flush=True)
            elif block.type == "text":
                print("\n[Response] ", end="", flush=True)
        elif event.type == "content_block_delta":
            delta = event.delta
            if delta.type == "thinking_delta":
                print(delta.thinking, end="", flush=True)
            elif delta.type == "text_delta":
                print(delta.text, end="", flush=True)
            elif delta.type == "input_json_delta":
                print(delta.partial_json, end="", flush=True)

    response = stream.get_final_message()
```

`input_json_delta` がツール引数 JSON の断片として流れてきて、`stream.get_final_message()` を呼んだ後でようやく完成形の `block.input` を取れる、というあたりは、ライブで見ないと挙動が想像できなかった部分だ。

## Skills まわりで現地で交わされた Q&A

Skills のパートは質疑が最も活発で、現場で運用している人たちのリアルな悩みが出てきた。要点だけ箇条書きで残しておく。

- **Q. スキルの品質をどう測るか／公式ベンチマークはあるか？**
  A. 公開ベンチマークは無い。Anthropic 公式の skill-writing skill に書かせて Claude 自身に評価させるのが現実解。可能なら eval を当て、A/B testing で改善前後を比較する。
- **Q. スキルの marketplace はあるか／落としてきて使ってよいか？**
  A. 存在する（Claude Code UI / desktop UI 経由）が、**セキュリティ脆弱性** が前面に立つ論点。OSS 由来のスキルは必ずレビューしてから使い、更新が走るので「最後に security review した日時」を運用記録として持っておく。
- **Q. スキルの再利用と version control はどうしている？**
  A. **スライスする** のが鉄則。「RFP 応答スキル」のような巨大スキルを作らず、interpretation / vulnerability / brand design のように単機能に切る。バージョン管理＋ A/B testing で「3 つ前の iteration がいちばん良かった」に戻れるようにしておく。
- **Q. スキル同士の矛盾やハルシネーションをどう検出する？**
  A. Eval を用意して引っかけるのが基本線。ただし「どのスキルが原因か」までは追いづらい。Claude は他モデルよりも「分からない」と言いやすく設計されているので、エージェント側に **「分からなければ human にエスカレーション」** の足場を組んでおく。1M トークンの context があるからといって全部使う必要はない、というのも繰り返された注意点。
- **Q. スキル内でツール／手順を明示すべきか、モデルの判断に任せるか？**
  A. **明示してよい**。スクリプトをスキル内に置いて「ここでこれを実行せよ」と書いている運用例もある。Skills は知識層であり、その下で MCP / API / Plugins を動かすための **オーケストレーション指示** を書ける場所だと位置づけるのがしっくり来る。

最後に一つ、Q&A の流れで講師がこぼした一言を残しておきたい。「モデルが新しくなるたびに、CLAUDE.md やスキルを見直す癖を持つこと」 ── 1 年前のプロンプトは 1 年前のモデル向けに最適化されている。「役割を与える」「最も賢い存在として振る舞え」のような古い手癖は、今のモデルではむしろ不要、あるいはノイズになる。**スキルは書いて終わりではなく、モデルが進化するたびに読み返す対象** だ、というのが妙に印象に残った。

## 現場に持ち帰りたいこと

### 3 層責務モデルとして指示の置き場所を分ける

ハンズオンを通して一番強く残ったのは、エージェントの挙動を決める指示には **3 つの置き場所** があり、それぞれに固有の責務がある、という考え方だった。

| 層 | 何を書くか |
|---|---|
| **System Prompt** | エージェントのロール・SLA・エスカレーション基準 |
| **Tool description** | そのツールを **いつ** 呼ぶか、入力の妥当性 |
| **Tool 実装** | 安全網。不正入力の弾き返し、ヒットなし時のフォールバック |

すべてを system prompt に詰め込んでいる実装をよく見るけれど、ツール選択の不具合は tool description に書くべきだし、「ヒットなし=エスカレーション候補」のような **次の手のヒント** はツール実装側の戻り値に埋めるべきなのだ、と整理がついた。講師が「ほとんどの AI システム障害はモデル問題ではなく prompt・agent 構成・tool 設計の問題である」と何度も繰り返していたのが、ようやく自分のものとして腑に落ちた。

### `effort=high` と `effort=low` を並べて投げる、をデバッグ手段にする

同じチケットを `high` と `low` で実行し、思考トレースを横に並べると、Claude が **何を見て何を判断したか** が驚くほど読める。`high` でしか拾えない仮説があるなら、その種のチケットだけ `high` にルーティングすればよい。これは triage の本番設計だけでなく、エージェント開発中の **デバッグツール** としても優秀だと感じた。

## もっと深掘りする入口

- [Tool use 概要](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Agentic tool use パターン](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/agentic-tool-use)
- [Extended / Adaptive Thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [Structured Outputs (JSON Schema)](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Streaming Messages](https://docs.anthropic.com/en/docs/build-with-claude/streaming)
- [Model Context Protocol — 仕様サイト](https://modelcontextprotocol.io/)
- [Claude Skills のドキュメント](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/overview)
- [Claude API Documentation](https://docs.anthropic.com/en/docs)

## 章末 — シングルエージェントの足場を踏み固めること

第1章で受け取った takeaway のひとつに、「マルチエージェント評価は未解決問題である」というものがあった。Anthropic 自身も orchestrator-subagent の振る舞いをどう測るかについて、まだ決定版を持っていない。

その未解決領域に踏み込む前に、**シングルエージェントを自分の手で組める** という足場が要る、というのがこの章を通じて私が腹落ちしたことだ。ループの輪郭、thinking の連続性、format のスコープ、`effort` のコスト感。これらの肌触りを知らないままマルチエージェントに進むと、起きている問題がモデルの問題なのか、ループの問題なのか、ツール設計の問題なのか、永遠に切り分けられない。

フレームワークは便利だ。けれど、いちど **フレームワーク無しでループを書く** という経験を通っておくと、後でどんな抽象を被せても、その下で何が起きているかが見えるようになる。冒頭で見た 5 つの足場のうち API と MCP は下層、Skills は知識層、Plugins はそれらを束ねる入れ物 ── という **層の自覚** を持って Messages API のループを眺めると、どの問題をどの層で解くべきかの判断がぐっと早くなる。

次章ではこの足場の上に、**プロンプト自体を評価駆動で直していく** という工学的プロセスを乗せる。`/04-prompt-rescue` に進もう。

→ [第4章 Prompt Rescue — 評価駆動でプロンプトを直す](./04-prompt-rescue)
