---
title: "推論最適化 — TTFT / TTC / OTPS / Cost を測って効かせる"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day2/02_inference-optimization/`

## はじめに — p99 で語るということ

SF 滞在 2 日目、午後の Inference Optimization のセッション。スピーカーがホワイトボードに線を一本引いて、こう言った。「平均レイテンシで SLA を語るのはやめてくれ。あなたのクライアントは平均では生きていない」。

そのとき初めて、自分が今まで「平均で速い」を「速い」と言い換えていたことに気づいた。ダッシュボードに並んでいた `avg(latency)` の数字を、何の疑問もなく週次報告に貼って、緑色のままにしていた。

レストランの比喩が出てきたのはその直後だった。半分の客には 5 分で料理が出てきて、半分の客には 25 分かかる店。平均は 15 分。これを「平均 15 分の店です」と紹介されて納得する客はいない。実際に評判を決めるのは、運悪く 25 分待たされた側の声だ。

p99 で語る。これがこの章で持ち帰った最大の言葉になった。あとの話はすべて、その一言の補強材料だったように思う。

---

## 題材 — 5 ステージと 4 指標

ハンズオンの題材は、Anthropic API への単発呼び出しから始めて、最終的にプロンプトキャッシング付きのマルチターン会話までを Notebook 上で計測する `Inference_Optimization.ipynb`。動かしながら何度も Cell を再実行し、数字が変わる様子を眺めるタイプの章だった。

LLM のレスポンスは、ユーザの目には 1 本のストリームに見える。だが内部では明確に 5 つのステージに分かれている。

```
[1] Prompt 送信
   ↓ (ネットワーク)
[2] API Gateway / Routing
   ↓
[3] Tokenization
   ↓
[4] Inference (prefill → decode)
   ↓ (ここで最初のトークンが出る = TTFT)
[5] Streaming Response
   ↓ (最後のトークン到達 = TTC)
```

そして、観測すべき指標は次の 4 つ。

- **TTFT** (Time To First Token): 最初のトークンが返るまで。Consumer 体感を決める。
- **TTC** (Time To Completion): すべての出力が揃うまで。Builder のスループットと SLA を決める。
- **OTPS** (Output Tokens Per Second): `output_tokens / (TTC - TTFT)`。生成フェーズだけの純粋な速度。
- **Cost**: 入出力トークン数 × モデル単価。請求書に直結する数字。

ここで OTPS の式が `output_tokens / TTC` ではなく `output_tokens / (TTC - TTFT)` であることを、講師がしつこいくらい強調した。TTC で割ると prefill 待ち（waiting）が混ざるので「実効スループット」になる。decode の本当の速さを見たいなら、最初の 1 トークンが出てからの時間で割らないといけない。これは知らなかった。

---

## 何を学んだか — 数字の輪郭が変わっていく

### 同じプロンプトでも、モデル選択で景色は一変する

最初の比較実験で、同じ `"What is machine learning? Answer in 2 sentences."` を Haiku / Sonnet / Opus にそれぞれ 5 回ずつ流した。

| Model  | Runs | TTFT (ms) | TTC (ms) | OTPS  | $/1K calls |
|--------|------|-----------|----------|-------|------------|
| haiku  | 5    | 585       | 866      | 189.4 | $0.3463    |
| sonnet | 5    | 1504      | 2297     | 65.2  | $4.0800    |
| opus   | 5    | 800       | 1379     | 87.1  | $20.1000   |

数字の意味を読み解く時間、しばらく無言になった。コストは Haiku から Opus へ約 60 倍。TTFT は Haiku が圧倒的に速く、しかし Opus は Sonnet よりも速いという逆転すらある。「賢いから Opus」「安いから Haiku」と短絡的に選んでいた自分の判断軸は、この一枚の表で揺らいだ。

### p99 を見る、ということ

セッションの後半、講師が「全試行の生データを残せ」「`mean()` だけで判断するな」と何度も言った。

ベンチマークの結果を 1 行に圧縮するのは楽だ。でもそれをやった瞬間、`p95`, `p99` を後から見る権利を捨てている。Notebook の `BenchmarkResult` データクラスはまさにこの設計で、すべての試行を個別に保持して、後から分位点を出せるようになっていた。実装の段取りそのものが「p99 で語るための準備」になっている。

### Prompt Caching の経済モデル — 倍率で覚える

価格を絶対額で覚えると、モデルが変わるたびに頭がリセットされる。倍率で覚えるとずっと楽だ、というのも気づきだった。

| 操作 | 単価 | ベース倍率 |
|---|---|---|
| 通常入力 | $3.00 | 1.0× |
| キャッシュ書込（5min TTL） | $3.75 | 1.25× |
| キャッシュ書込（1h TTL） | $6.00 | 2.0× |
| キャッシュ読出 | $0.30 | 0.1× |
| 出力 | $15.00 | (変動なし) |

損益分岐の計算もシンプル。

- **5min cache**: 1 回書いて 2 回読めば、1.25× + 0.1×2 = 1.45× < 普通に 2 回叩く 2.0×。→ **2 回目の読み出しで元が取れる**。
- **1h cache**: 1 回書いて 3 回読めば、2.0× + 0.1×3 = 2.3× < 3.0×。→ **3 回読めば明確に得**。

「同じプレフィックスを 2 回以上叩く見込みがあるなら、5min cache は基本入れ得」。これがその場で取った最短のメモになった。

### Tool use のラウンドトリップが TTFT を倍にする

Calculator tool を呼ばせる版と呼ばせない版を比較した結果。

| 条件 | TTFT (avg) | TTC (avg) |
|---|---|---|
| Without tool | 1437ms | 3250ms |
| With tool    | 2339ms | 3910ms |

TTFT がほぼ +900ms。原因は明白で、1 回目の呼び出しで `tool_use` ブロックを受け取り、ローカルで実行して、結果を持って 2 回目を呼ぶ往復が入るから。エージェントを触っていて「なんか遅い」と感じる原因の大半はここだ、と腹落ちした瞬間だった。ツールを増やすほどラウンドトリップが膨らむ。これも数字で確認できると、設計の議論で説得力が違う。

### Opus 4.7 のトークナイザ変更 — eval を回す判断

Opus 4.7 では新しいトークナイザが入っている。同じテキストでも最大 30% 程度トークン数が増えるケースがあると聞いた。

これは「単価が下がったから移行」という短絡的な判断を許さない数字だった。プロンプトによっては、単価減少分をトークン増加が食いつぶす。判断するには、自分のプロンプトで eval を回すしかない。`usage.input_tokens` / `usage.output_tokens` の合計を、モデルごとに並べて比較する。一手間だが、これをやらないと議論のスタートラインに立てない。

---

## 前提が崩れた瞬間

セッションの中で、自分の中の前提が静かに崩れた瞬間がいくつかあった。

**「1M context があるからキャッシュは要らない」と思っていた**。これは間違いだった。1M 入るのと、1M を毎回 prefill するのは別の話だ。同じプレフィックスを 2 回以上叩く想定があるなら、context window の広さに関係なく入れ得。むしろ、長いほど効く。

**「タイムスタンプを system prompt に入れていた」**。何の気なしに `f"Current time: {datetime.now()}..."` と書いていたが、これは毎リクエスト 1 文字違うので、永遠に cache miss する。動的な値はプレフィックス側に置いてはいけない。動的部分は user message の末尾に寄せる。これは即座に修正項目になった。

**「1024 トークン未満の `cache_control` にも意味がある」と思っていた**。Sonnet / Opus は 1,024 トークン以上、Haiku は 4,096 トークン以上のブロックでないとキャッシュされない。これ未満で `cache_control` を付けても、エラーにはならず黙って無視される。`usage.cache_creation_input_tokens` が 0 のまま「効いているはず」と信じ続けることになる。これは事故ったら気づきにくい。

---

## 押さえておきたいコード／設定

### Streaming で TTFT を計測する

`client.messages.create()` は最終応答が揃ってから返るので TTFT が測れない。必ず `client.messages.stream()` を使い、`content_block_start` イベントが立った瞬間を TTFT として記録する。

```python
import time
import os
import anthropic

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


def _stream_request(prompt, model="claude-sonnet-4-5-20250929", max_tokens=256):
    """Stream a request and return (ttft, total_time, final_response)."""
    ttft = None
    start_time = time.perf_counter()

    with client.messages.stream(
        model=model,
        max_tokens=max_tokens,
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        for event in stream:
            if ttft is None and event.type == "content_block_start":
                ttft = time.perf_counter() - start_time
        response = stream.get_final_message()

    total_time = time.perf_counter() - start_time
    return ttft, total_time, response


def compute_otps(ttft, total_time, output_tokens):
    """waiting を除いた純粋な generation speed を返す。"""
    gen_time = total_time - ttft
    return (output_tokens / gen_time) if gen_time > 0 else 0
```

`time.perf_counter()` を使っているのも意図がある。`time.time()` は壁時計で NTP 同期によるジャンプがあり得るので、ミリ秒単位のレイテンシ計測には向かない。

### `cache_control` を system block に打つ

system prompt は通常 string で渡すが、キャッシュを使う場合は content block の list に変える必要がある。そして、効いていることを `usage` で確認するところまでがセット。

```python
SYSTEM_PROMPT = LONG_INSTRUCTIONS  # 1,024 トークン以上であること

system_block = [
    {
        "type": "text",
        "text": SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},
    }
]

response = client.messages.create(
    model="claude-sonnet-4-5-20250929",
    max_tokens=256,
    system=system_block,
    messages=[{"role": "user", "content": question}],
)

# 効いているか必ず確認する
print(f"cache_creation_input_tokens: {response.usage.cache_creation_input_tokens or 0}")
print(f"cache_read_input_tokens:     {response.usage.cache_read_input_tokens or 0}")
print(f"input_tokens (uncached):     {response.usage.input_tokens}")
```

- **初回**: `cache_creation_input_tokens > 0`, `cache_read_input_tokens == 0`
- **2 回目以降（5min 以内）**: `cache_creation_input_tokens == 0`, `cache_read_input_tokens > 0`

2 回目以降で `cache_read_input_tokens` が 0 のままなら、プレフィックスのどこかが揺れているか、最低トークン数に届いていないか、のどちらか。

### マルチターン会話での breakpoint 切替

マルチターンでキャッシュを効かせるコツは、**直前の assistant ターンだけに `cache_control` を残し、それより古いターンの cache_control は平文に戻す**こと。breakpoint は積み上げない、最新だけに移す、という運用。

```python
SYSTEM = [
    {
        "type": "text",
        "text": SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},
    }
]


def chat(messages, new_question):
    # 1. 古い breakpoint を平文に戻す
    for msg in messages:
        if msg["role"] == "assistant" and isinstance(msg["content"], list):
            msg["content"] = msg["content"][0]["text"]

    # 2. 最新の assistant ターンを breakpoint にする
    if messages and messages[-1]["role"] == "assistant":
        messages[-1]["content"] = [
            {
                "type": "text",
                "text": messages[-1]["content"],
                "cache_control": {"type": "ephemeral"},
            }
        ]

    # 3. 新しい user メッセージを足してリクエスト
    messages.append({"role": "user", "content": new_question})

    response = client.messages.create(
        model="claude-sonnet-4-5-20250929",
        max_tokens=300,
        system=SYSTEM,
        messages=messages,
    )

    answer = response.content[0].text
    messages.append({"role": "assistant", "content": answer})
    return answer, response.usage
```

実測（5 ターン会話）では、ターンが進むごとに `cache_read` が積み上がっていく様子が、数字としてきれいに見える。

```
Turn 1: 4927ms | cached: 0    | created: 1241   ← Cold (system prompt 書込)
Turn 2: 5127ms | cached: 1241 | created: 319    ← system 再利用 + Turn1 ペア書込
Turn 3: 5156ms | cached: 1560 | created: 314    ← Turn1+2 まで再利用
Turn 4: 4615ms | cached: 1874 | created: 316
Turn 5: 6498ms | cached: 2190 | created: 316
```

新しい質問が来るたびに、過去の会話履歴全体を 0.1× の単価で再処理できる。ターン数が増えてもコストが線形に伸びない。これが見えると、マルチターンエージェントの設計判断が変わる。

---

## 現場に持ち帰りたいこと

セッションを終えて、明日からの仕事で変えるべきことが 3 つ整理できた。

**1. SLA 議論の場で「平均」と言わない**。p95 / p99 で語る。`mean()` だけ書いてあるダッシュボードは、まず分布が取れる形に作り直す。Notebook で `BenchmarkResult` を全試行残していたあの設計を、本番の計測にも持ち込む。

**2. キャッシュ検証は `usage.cache_creation_input_tokens` / `usage.cache_read_input_tokens` で必ず観測する**。「効いてるはず」で済ませない。CI で同一プロンプトを 2 回叩いて、2 回目の `cache_read_input_tokens > 0` をアサートするテストを入れておけば、誰かが system prompt にタイムスタンプを足した瞬間に気づける。気づけないと請求額が静かに 4 倍になる。

**3. AWS Bedrock / Strands SDK との互換性**。`cache_control` の構造は Bedrock 経由でも同じで、Strands なら `CacheConfig(strategy="auto")` で SDK 任せにできる。Bedrock 経由では `usage` のフィールド名が `cacheWriteInputTokens` / `cacheReadInputTokens` のキャメルケースになるので、ログ出力やダッシュボードは両方に対応させておく。

```python
# Bedrock 経由（Anthropic SDK）
client = anthropic.AnthropicBedrock(aws_region="us-west-2")

response = client.messages.create(
    model="anthropic.claude-sonnet-4-5-20250929-v1:0",
    max_tokens=256,
    system=[{
        "type": "text",
        "text": LONG_SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral", "ttl": "1h"},
    }],
    messages=[{"role": "user", "content": question}],
)
```

```python
# Strands Agents SDK（Bedrock）
from strands import Agent
from strands.models import BedrockModel
from strands.models.bedrock import CacheConfig

model = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20250929-v1:0",
    region_name="us-west-2",
    cache_config=CacheConfig(strategy="auto"),
)

agent = Agent(
    model=model,
    system_prompt=LONG_SYSTEM_PROMPT,
    tools=[calculator, current_time],
)
```

Strands の旧 API `cache_tools="default"` は deprecation 方向（Issue #1577）。新規実装は `cache_config` を使う。

---

## もっと深掘りする入口

セッションで紹介された一次資料のうち、特に手元に置いておきたいもの。

- **Prompt Caching**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- **Streaming Messages**: https://docs.anthropic.com/en/api/messages-streaming
- **Token Counting API**: https://docs.anthropic.com/en/api/messages-count-tokens
- **Message Batches API**: https://docs.anthropic.com/en/docs/build-with-claude/batch-processing — オフライン処理 50% 引き
- **AWS Bedrock prompt caching**: https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
- **Strands Agents SDK**: https://strandsagents.com/
- **OpenTelemetry GenAI semantic conventions**: https://opentelemetry.io/docs/specs/semconv/gen-ai/ — TTFT / TTC を分散トレースに乗せたい場合の標準

---

## 章末 — 工学責任と次章へ

第 1 章で「IT の役割は変わる」という takeaway を書いた。コードを書く人から、判断の輪郭を引く人へ、と。

この章を経て、その輪郭にもう一本線が足された気がしている。クライアントとの議論の場で「動きます」と言う代わりに、「p99 でこれを満たします」「キャッシュ命中率がこの値を割ったらアラートを上げます」と言える人になる。動くかどうかではなく、どの確率で・どの遅さで・いくらで動くか。それを数字で約束できる立場を、自分から取りに行く。

これが、今回学んだ「p99 で語る」という言葉の、もう一つの意味だと思う。

次章では、推論最適化と表裏一体の **Context Engineering** に踏み込む。速く・安く回せるようになった土台の上に、どのような情報を・どの順序で・どの形式で渡せばモデルが間違えないか。`Context Engineering` の Notebook を通して、`day2/03_context-engineering/` の体験を整理する。

→ [08-context-engineering](./08-context-engineering)
