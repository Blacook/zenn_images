---
title: "Developer Platform — Messages API でエージェントを素手で組む"
free: true
---

> **リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day1/02_developer-platform/` (`Developer_Platform.ipynb`)
> **モデル**: `claude-sonnet-4-6`

## はじめに — 「Agent SDK の下」を一度だけ自分の手で組む

Anthropic Developer Platform は、Agent SDK、Claude Code、Batch API、Files API までを含む大きな面だが、この章ではその最下層にあたる **Messages API** だけを扱う。題材は TechFlow（中堅 B2B SaaS）の Tier 1 サポート triage エージェントで、tool use・adaptive thinking・structured output・streaming の 4 要素を **フレームワークを噛まさず** に 1 本のループへ組み上げる。SDK で隠れている境界線が見えると、上層がなぜそう設計されているのか、どこを触ると壊れるのかが見える ── というのが本章のねらいである。

## 題材 — TechFlow Tier 1 自動化の概要

TechFlow は中堅 B2B SaaS で、Tier 1 サポートが 1 日 500 件超 (`500+ tickets/day`) のチケットを 1 件あたり約 8 分で処理している。これを Claude Sonnet 4.6 に置き換え、Tier 2 へのエスカレーション以外を自動化するのが Build-Along の目的である。

エージェントが触れるのは 3 つのツールだけ。

| ツール           | 役割                                                          |
| ---------------- | ------------------------------------------------------------- |
| `get_ticket`     | ID から本文・顧客・優先度・プロダクト領域を取得               |
| `search_kb`      | キーワードで Knowledge Base を検索 (最大 3 件)                |
| `resolve_ticket` | 解決ノートを書いて `resolved` / `escalated` / `closed` に遷移 |

データはモック。サンプルチケットは `TKT-1042`（重複請求）、`TKT-1043`（API キーローテーション後の webhook 失敗）、`TKT-1044`（バルクエクスポート要望）、`TKT-1045`（管理者 MFA ロックアウト、Critical）、`TKT-1046`（Singapore 拠点で API 間欠 500 エラー）の 5 件。Knowledge Base は `KB-001`〜`KB-007` の 7 本で、典型対応（重複請求の返金フロー、webhook 署名鍵の再生成、admin recovery など）と外し（バルクエクスポートはロードマップ）の両方が混ざる構成になっている。

道具立てを最小にすることで、ループ・思考・構造化・ストリームの 4 要素を組み上げる「外側」に意識を集中できる構成になっている。

## ベストプラクティス・アンチパターン・重要ポイント

### Agentic loop は `stop_reason` を軸に組む

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: エージェントの本体は「`response.stop_reason == "tool_use"` のあいだ回す while ループ」である。各イテレーションで `tool_use` ブロックを実装にディスパッチし、結果を `tool_result` で次ターンへ積む。`stop_reason` の取りうる値は 3 つで、`"tool_use"` ならループ継続、`"end_turn"` で終了、`"max_tokens"` は実質エラーとして扱う。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**: ループ脱出条件をターン数や時間に置く実装。Claude は途中で「もう一段ツールを呼びたい」と判断する場合があり、`stop_reason` 以外で打ち切ると tool call が宙に浮き、次ターンに整合性エラーを返す。

</div>

**ハンズオンでの具体例**: ノートブック Part 1 の `run_agent()` は、`max_tokens=32000`・`thinking={"type": "adaptive"}` を全コールに渡し、`while response.stop_reason == "tool_use":` 直下で `tool_result` を組み立てる。`tool_result` の `tool_use_id` には **`block.id`（受信した `ToolUseBlock` の id）** を必ず入れる。これがないと API がエラーで弾く。

### 思考モードと effort は粒度を分ける

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: `thinking` パラメタは「思考をどう生成するか」、`output_config.effort` は「どこまで深く考えるか」を制御する。`thinking` の値は `"adaptive"`（複雑さに応じて自動）／`"enabled"`（常に生成）／`"none"`（無効）の 3 種。`effort` は `"low"` / `"medium"` / `"high"` / `"xhigh"` / `"max"` の 5 段階で、**API デフォルトは `"high"`**（パラメータ省略時と同じ挙動）。`xhigh` は Claude Opus 4.7 専用の拡張レベルで、長時間のエージェント・コーディングタスク向けに Anthropic は **Opus 4.7 のコーディング／エージェント用途は `xhigh` から始めることを推奨**している。`max` は Mythos Preview / Opus 4.7 / Opus 4.6 / Sonnet 4.6 で利用可能な絶対最大の能力。adaptive + effort の組み合わせで「複雑なら深く、簡単なら浅く、上限はこちらで握る」が成立する。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**: 本番系で常に `effort="max"` や `xhigh` を貼る。レスポンス時間とトークン課金が数倍〜十数倍に膨らみ、500 件/日のスケールで経済合理性が崩壊する。逆に、曖昧チケットを `low` に固定すると、必要な仮説立てが行われずミスエスカレーションが増える。`high` をデフォルトと知らずに「念のため `xhigh`」を貼るのも、Sonnet 系では無効な値で実行時エラーになる罠がある（`xhigh` は Opus 4.7 限定）。

</div>

**ハンズオンでの具体例**: Part 2 の `run_agent_thinking()` は同一チケット `TKT-1046`（Singapore 拠点・15% の API が間欠 500）を `high` と `low` で連投する。`high` の思考トレースには「Singapore region routing degradation の可能性」「retry-success の頻度が rate limit パターンと一致しない」といった仮説が立つ一方、`low` ではほぼ言及されない。経過時間も `high≈22s` / `low≈19s` と差が出る。Opus 4.7 で実運用に持ち上げるなら、複雑チケットを `xhigh`、単純チケットを `medium`〜`low` にルーティングするのが筋の良い設計になる。

### `format` (JSON schema) は最終 call でのみ使う

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: `output_config.format` を渡すと、Claude の **すべてのテキスト出力が JSON Schema に制約される**。これが効くのはツールループが完了したあとの最終呼び出しに限られる。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**: ループ中の API コールに `format` を入れる。`tool_use` を返したいタイミングで「テキストは JSON でなければならない」と引っ張られ、ツール呼び出しが壊れる。スキーマで `additionalProperties` を省略するのも罠で、Claude が定義外のフィールドを生やして下流のパースが死ぬ。

</div>

**ハンズオンでの具体例**: Part 1 後半の `run_agent_structured()` は、ツールループ中は `output_config` を渡さず、ループ後に `messages` へ "Provide your structured resolution as JSON." を追加し、**1 度だけ** `output_config={"format": RESOLUTION_SCHEMA}` と `tool_choice={"type": "none"}` を併用して構造化 JSON を取り出す。スキーマには `additionalProperties: False` を必ず入れる。`tool_choice` の取りうる値は `none` / `auto` / `any` / `{"type": "tool", "name": ...}` の 4 種で、最終 call では `none` を明示する。

### Tool description は呼ばれ方を決める

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: `tool` スキーマの `description` は Claude にとっての API ドキュメントで、`name` と `input_schema` 以上に「いつ呼ぶか／呼ばないか」を左右する。`input_schema` の各プロパティ `description` まで具体例を書くと、入力の質も上がる。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**: 「Search the knowledge base.」のような事務的な 1 行で済ませる。Claude は「順序」「前提」「不可ケース」を読み取れず、`get_ticket` を飛ばして `search_kb` をいきなり呼ぶ、解決前に `resolve_ticket` を打つ、といった経路を取りはじめる。

</div>

**ハンズオンでの具体例**: ノートブックの `search_kb` description は "Use this to find troubleshooting steps, policies, or known solutions **before attempting to resolve a ticket.**" と **順序のヒント** が埋まっている。`resolve_ticket` の `status` プロパティは `enum: ["resolved", "escalated", "closed"]` と値域を絞り、description で `resolved = fix applied; escalated = needs Tier 2; closed = duplicate or invalid` と意味を明示している。これにより Claude のツール選択がほぼブレない。

### Content block を取りこぼさない

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: `response.content` は `ThinkingBlock` / `ToolUseBlock` / `TextBlock` の混在リストで返る。assistant ターンを次ターンへ積み戻すときは **`response.content` をそのまま** 渡す。extended thinking はサーバ側で連続性を検証しているため、`ThinkingBlock` を落とすと整合性エラーになる。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**: `text` だけ・`tool_use` だけを抽出して `messages` に積む。adaptive thinking 有効時には必ず壊れる。また、最終の構造化 JSON を取り出すときに `content[0]` を見る実装も罠で、`[ThinkingBlock, TextBlock]` の並びだと JSON は **末尾の TextBlock** に入る。

</div>

**ハンズオンでの具体例**:

```python
# NG: thinking を捨てて積み戻す
filtered = [b for b in response.content if b.type != "thinking"]
messages.append({"role": "assistant", "content": filtered})

# OK: そのまま渡す
messages.append({"role": "assistant", "content": response.content})

# 構造化 JSON は末尾の TextBlock から取る
text_blocks = [b for b in response.content if b.type == "text" and b.text.strip()]
data = json.loads(text_blocks[-1].text)
```

`ToolUseBlock` の `block.id` は対応する `tool_result.tool_use_id` と一対一で紐付く。`block.name` / `block.input` (dict) でツール呼び出し本体を取り出す。

### Streaming は UX 改善メトリクスである

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: `client.messages.stream()` はコンテキストマネージャで、`content_block_start` / `content_block_delta` / `content_block_stop` の 3 種イベントが順番に流れる。`content_block_delta` の `delta.type` は `thinking_delta`（思考トークン）／`text_delta`（応答トークン）／`input_json_delta`（ツール引数 JSON の断片）の 3 種で、これらを別々にハンドリングすると思考・応答・ツール引数が独立した流れとして可視化できる。ストリーム終了後は `stream.get_final_message()` で完全な `Message` オブジェクトを取得する。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**: ストリーミングをトークン節約や速度短縮の手段だと誤解すること。実体は UX 上の体感改善 ── 「2 段落待たせる」のではなく「1 文ずつ流れる」 ── であって、レイテンシ自体は短くならない。また、ストリーム中に `block.input` を読もうとしても、ツール引数 JSON は `get_final_message()` 呼び出し後に確定するため、ストリーム終了前のアクセスは不完全な値を返す。

</div>

**ハンズオンでの具体例**: Part 3 の `run_agent_streaming()` では `content_block_start` で `block.type` を見て `[Thinking]` / `[Tool: name]` / `[Response]` のラベルを切り替え、`content_block_delta` の 3 種 delta をそれぞれ `flush=True` で書き出す。最終構造化出力もストリームで取り、`text_delta` だけを画面に流す。

### クライアント側の地味な落とし穴

<div style="background:#f0faf3;border-left:4px solid #1a7f37;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**原則**: 「Messages API はサーバ側に状態を持ち、SDK は薄い HTTP クライアントである」と考えて構成パラメタを揃える。

</div>

<div style="background:#fff5f5;border-left:4px solid #b42318;padding:0.75em 1em;margin:1em 0;border-radius:4px;color:#1f2328;">

**アンチパターン**:

- `timeout` を省略する。`max_tokens > 21333` で非ストリーミング呼び出しをすると、SDK デフォルトのタイムアウトに先に当たって落ちる。
- API キーをノートブックや Git にハードコードする。
- 1 年前のプロンプト・スキル定義を新モデルに使い回す（旧モデル向けの「最も賢い存在として振る舞え」「役割を与える」式のメタ指示は、新モデルではノイズになる場合がある）。

</div>

**ハンズオンでの具体例**: ノートブック冒頭で `anthropic.Anthropic(timeout=900.0)` と明示している。API キーは `os.environ["ANTHROPIC_API_KEY"]` 経由で読み、`client.messages.create()` の `model` には `MODEL = "claude-sonnet-4-6"` を渡す。Setup セルは接続確認 (`Reply with only: ready`) と SDK バージョン表示を兼ねており、これを毎回最初に走らせるとデバッグが楽になる。

## 押さえておきたいコード／設定

### クライアント初期化

```python
import os
import anthropic

client = anthropic.Anthropic(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    timeout=900.0,  # max_tokens を大きく取る場合に必要
)
MODEL = "claude-sonnet-4-6"
```

### ツール schema（順序ヒントを description に埋める）

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

        # ThinkingBlock を含む content をそのまま積み戻す
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

### Streaming の受信ループ

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

### 構造化出力の最終 call

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

messages.append({"role": "user",
                 "content": "Provide your structured resolution as JSON."})

final = client.messages.create(
    model=MODEL,
    max_tokens=8000,
    system=SYSTEM_PROMPT,
    output_config={"effort": effort, "format": RESOLUTION_SCHEMA},
    tool_choice={"type": "none"},
    thinking={"type": "adaptive"},
    messages=messages,
)

text_blocks = [b for b in final.content if b.type == "text" and b.text.strip()]
data = json.loads(text_blocks[-1].text)
```

## よくある勘違いと気づき

- **「思考ブロックは next call に渡さなくていい」と思っていた** が、これは違った。`ThinkingBlock` は `text` や `tool_use` と一緒に **そのまま** 次の `messages` に積まないと整合性エラーで弾かれる。サーバ側に「思考の連続性」という状態があると知って、ようやく adaptive thinking の手触りが掴めた。

- **「`effort="high"` にしておけば安心」と思っていた** が、これも違った。`high` は確かに深く考えてくれるが、レスポンス時間とトークン課金が膨らむ。請求の二重課金のような単純チケットは `low` で十分捌け、API 間欠障害のような曖昧チケットでのみ `high` に振る、というルーティングが現実解になる。`TKT-1046` を `low` と `high` で並べて投げた瞬間、`high` 側のトレースに "Singapore region routing issue" の仮説が立っているのを見て、「これは確かに `high` が要る」と腹落ちした。

そして workshop の素材では `high` / `medium` / `low` の 3 段階で説明されていたが、章をまとめる段でドキュメントを引き直してみると、実は **`low` / `medium` / `high` / `xhigh` / `max` の 5 段階** が正しい仕様だった。**API デフォルトは `high`**（パラメータ省略時と同じ）で、`xhigh` は Opus 4.7 専用の拡張レベル、`max` は Opus 4.6 / 4.7 と Sonnet 4.6 で使える絶対最大。Anthropic は Opus 4.7 のコーディングや長時間のエージェントタスクでは `xhigh` から始めることを推奨している。3 段階で思考停止していると、最も「考えさせる」ための引き出しを 2 つも見逃していたわけで、モデル世代と一緒に effort の階層ごと読み直す対象なのだと、ここでもう一度突きつけられた。

- **「`output_config.format` は最初から付けっぱなしでよさそう」と思っていた** が、一番派手にハマるパターンらしい。`format` は **すべてのテキスト出力に効く** ので、ツールループ中に有効化すると `tool_use` を返したいタイミングで Claude が引っ張られ、ツール呼び出しが壊れる。**ループ中は `effort` だけ、最終 call でだけ `format` + `tool_choice={"type": "none"}`**、というのが鉄則になった。

- **「`response.content` の最初の text ブロックを読めばよい」と思っていた** が、これも罠だった。adaptive thinking 有効時は `[ThinkingBlock, TextBlock, ...]` の並びで返り、JSON は **最後の** TextBlock に入る。`content[0]` を見にいくと、`thinking` の手前にある中間テキストを掴んでしまうことがある。

> Q&A の流れで一つ印象に残った言葉がある。**「モデルが新しくなるたびに、CLAUDE.md やスキルを見直す癖を持つこと」** ── 1 年前のプロンプトは 1 年前のモデル向けに最適化されている。「役割を与える」「最も賢い存在として振る舞え」のような古い手癖は、今のモデルではむしろノイズになる。Messages API のループも同じで、SDK バージョンと一緒に **`thinking` / `output_config` / `tool_choice` の前提を毎リリース読み直す** 対象だと思うようになった。

## 現場に持ち帰りたいこと

- **3 層責務モデルで「指示の置き場所」を分ける。** エージェントの挙動を決める指示は 3 つの層に分けると整理しやすい。

| 層                   | 何を書くか                                               |
| -------------------- | -------------------------------------------------------- |
| **System Prompt**    | エージェントのロール・SLA・エスカレーション基準          |
| **Tool description** | そのツールを **いつ** 呼ぶか、入力の妥当性               |
| **Tool 実装**        | 安全網。不正入力の弾き返し、ヒットなし時のフォールバック |

すべて system prompt に詰め込む実装が多いが、ツール選択の不具合は tool description に書くべきだし、「ヒットなし=エスカレーション候補」のような **次の手のヒント** はツール実装の戻り値（KB-000 の "Consider escalating to Tier 2 support." など）に埋めるべきだ。講師が繰り返した「ほとんどの AI システム障害はモデル問題ではなく prompt・agent 構成・tool 設計の問題である」という言葉は、この 3 層を意識すると具体的に効いてくる。

- **`effort=high` と `effort=low` を並べて投げる、をデバッグ手段にする。** 同じチケットを両方の effort で実行し、思考トレースを横に並べると、Claude が何を見て何を判断したかが読める。`high` でしか拾えない仮説があるなら、その種のチケットだけ `high` にルーティングする ── というのは triage の本番設計だけでなく、エージェント開発中のデバッグツールとしても優秀だった。

## もっと深掘りする入口

- [Tool use 概要](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Agentic tool use パターン](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/agentic-tool-use)
- [Extended / Adaptive Thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [Structured Outputs (JSON Schema)](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Streaming Messages](https://docs.anthropic.com/en/docs/build-with-claude/streaming)
- [Claude API Documentation](https://docs.anthropic.com/en/docs)

## 章末 — シングルエージェントの足場を踏み固めること

第 1 章で受け取った takeaway のひとつに「マルチエージェント評価は未解決問題である」というものがあった。その未解決領域に踏み込む前に、**シングルエージェントを自分の手で組める** という足場が要る。ループの輪郭、thinking の連続性、format のスコープ、`effort` のコスト感 ── これらの肌触りを知らないままマルチエージェントへ進むと、起きている問題がモデルなのかループなのかツール設計なのか、永遠に切り分けられない。

次章ではこの足場の上に、**プロンプト自体を評価駆動で直していく** という工学的プロセスを乗せる。

→ 次章: [04-prompt-rescue](./04-prompt-rescue.md)
