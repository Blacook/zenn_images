---
title: "Day2-02: 推論最適化 — TTFT / TTC / OTPS / Cost を測って効かせる"
free: true
---

> **ハンズオン公式リポジトリ**: https://github.com/victorsteeb/Basecamp-Exercises.git
> **該当ディレクトリ**: `day2/02_inference-optimization/`

## はじめに — p99 で語るということ

SLA（サービス品質保証）を平均レイテンシで語ることは、クライアントが体験する「遅さ」を包み隠してしまう。レストランの比喩がわかりやすい。半分の客には 5 分で料理が出てきて、もう半分には 25 分かかる店を「平均 15 分の店」と紹介されても、納得する客はいない。評判を決めるのは運悪く 25 分待たされた側の声であり、SLA 議論でも同じ構造が起きる。

本章で扱う 4 指標と Prompt Caching の損益分岐は、その「運の悪い側」を数字で作り直すための道標である。`mean()` ではなく p95 / p99 を、`tokens / TTC` ではなく `tokens / (TTC - TTFT)` を、そして「速い/安い」ではなく「どの確率でどの遅さでいくらか」を語れるようにする。

:::details p95 / p99 とは？（パーセンタイルの読み方）
全リクエストを応答時間の短い順に並べたとき、下から何%目にあたる値かを示す指標。

| 指標        | 意味                    | 特徴                                                     |
| ----------- | ----------------------- | ------------------------------------------------------ |
| 平均（avg） | 全体の平均値            | 外れ値に引っ張られる。「遅いユーザー体験」を隠しやすい |
| **p95**     | 全体の95%がこの時間以内 | 高負荷時やエッジケースのユーザー体験を反映             |
| **p99**     | 全体の99%がこの時間以内 | 最も遅い1%のユーザー体験。SLAで使われることが多い      |

平均（mean）は外れ値に引っ張られるため、「遅い体験をしたユーザー」の存在を隠す。一方 p99 は「1000件に10件は必ずこれ以上かかる」という確率的な保証として使えるため、SLA（サービス品質保証）の定義に適している。
:::

---

## 題材 — 5 ステージと 4 指標

ハンズオンの題材は `Inference_Optimization`。単発呼び出しから始めて、モデル比較、Tool use ラウンドトリップ、Prompt caching の単発/マルチターン、最後に Bedrock / Strands 統合までを Notebook 上で計測する。

LLM のレスポンスはユーザの目には 1 本のストリームに見えるが、内部では 5 つのステージに分かれている。

```
[1] Prompt 送信
   ↓ (ネットワーク)
[2] API Gateway / Routing
   ↓
[3] Tokenization
   ↓
[4] Inference (prefill → decode)
   ↓ (prefill が終わって最初のトークンが出る = TTFT)
[5] Streaming Response
   ↓ (最後のトークン到達 = TTC)
```

観測すべき指標は次の 4 つ。

- **TTFT** (Time To First Token): 最初のトークンが返るまで。Consumer 体感を決める。
- **TTC** (Time To Completion): すべての出力が揃うまで。Builder のスループットと SLA を決める。
- **OTPS** (Output Tokens Per Second): `output_tokens / (TTC − TTFT)`。**分母は TTC ではない**。TTC で割ると prefill 待ちが混ざる。
- **Cost**: 入出力トークン数 × モデル単価。請求書に直結する。

:::message
**言語とトークン数**: 公式ドキュメント（[Pricing FAQ](https://platform.claude.com/docs/en/about-claude/pricing) / [Glossary](https://platform.claude.com/docs/en/about-claude/glossary)）によると、英語は 1 トークン ≈ 3.5〜4 文字だが、日本語・中国語などの CJK 文字は 1 文字あたり複数トークンを消費する。同一内容を日本語で書いた場合、英語版に比べてトークン数が **2〜3 倍**（＝コストも 2〜3 倍）になる。システムプロンプトやツール定義を英語で記述するだけで、単純計算で静的読み取りコストを **1/2〜1/3 に削減**できる。下のベンチマークも英語プロンプトで測定している点に注意。
:::

同一プロンプト `"What is machine learning? Answer in 2 sentences."` を Haiku / Sonnet / Opus に 5 回ずつ流したベンチマーク結果（各種平均）は以下のとおり。

| Model  | Runs | TTFT (ms) | TTC (ms) | OTPS  | $/1K calls |
| ------ | ---- | --------- | -------- | ----- | ---------- |
| haiku  | 5    | 585       | 866      | 189.4 | $0.3463    |
| sonnet | 5    | 1504      | 2297     | 65.2  | $4.0800    |
| opus   | 5    | 800       | 1379     | 87.1  | $20.1000   |

コストは Haiku → Opus で約 60 倍。TTFT は Haiku が最速、Opus が Sonnet より速いという順序の逆転も出ている。これらの 4 指標は Anthropic 直 API に加え、AWS Bedrock 経由・Strands Agents SDK 経由でも同様に測れる。

---

## ベストプラクティス・アンチパターン・重要ポイント

### クライアントには平均ではなく p99 を語る

:::message
**原則**: SLA は分布で語る。`mean()` は分散とテールを覆い隠す指標であり、本番でのユーザ体験は p95 / p99 が支配する。
:::

:::message alert
**アンチパターン**: ベンチマーク結果を集計値 1 行に圧縮して保存する。これをやった瞬間、後から分位点を取り出す権利を失う。
:::

#### **ハンズオンでの具体例**

Notebook の `BenchmarkResult` データクラスは全試行を個別に保持し、後段の `summary()` で `mean` を出す構造になっている。実装そのものに p99 を取り出すための処理が組まれている。「平均 15 分の店」のレストラン比喩は、p50 と p99 の差を意識するためのデフォルト言語として必ず持っておく。

-----

### TTFT / TTC / OTPS / Cost の 4 指標を粒度を分けて測る

:::message
**原則**: `TTFT` は Consumer 体感、`TTC` は Builder の SLA、`OTPS` は生成フェーズだけの純粋速度、`Cost` は請求書を決める。役割が違うので 1 つに丸めない。
:::

:::message alert
**アンチパターン**: OTPS を `output_tokens / TTC` で計算する。これは「実効スループット」になってしまい、prefill 待ち（waiting）が混ざる。decode の本当の速さを見るなら、最初の 1 トークンが出てからの時間で割る必要がある。
:::

#### **ハンズオンでの具体例**


```python
def compute_otps(ttft, total_time, output_tokens):
    gen_time = total_time - ttft  # ← 分母は TTC ではなく TTC − TTFT
    return (output_tokens / gen_time) if gen_time > 0 else 0
```

-----

### モデル選択は単純な性能比較ではない

:::message
**原則**: 「賢いから Opus」「安いから Haiku」という選び方では SLA を満たせない。プロンプトとユースケースごとに 4 指標を実測し、コストと TTFT / TTC のトレードオフを設計する。
:::

:::message alert
**アンチパターン**: ベンチマーク数値を見ずに「Opus は Sonnet より遅い」「Haiku が最速」と一般化する。実際には上記の表のように、TTFT で Opus が Sonnet より速いケースが起きる。
:::

#### **ハンズオンでの具体例**

大量バッチでコスト最優先なら Haiku、長い推論で品質最優先なら Opus、汎用なら Sonnet をデフォルトに置きつつ、`Inference_Optimization.ipynb` の Part 2 と同じ形で eval を回して自プロジェクトの分布を確認する。

-----

### 時間計測は `time.perf_counter()` を使う

:::message
**原則**: ミリ秒単位のレイテンシ計測には単調時計（monotonic clock）を使う。
:::

:::message alert
**アンチパターン**: `time.time()` を使う。これは壁時計で NTP 同期によるジャンプ（巻き戻し含む）があり得るため、レイテンシ計測には不向き。
:::

#### **ハンズオンでの具体例**

Notebook の `_stream_request` も `start_time = time.perf_counter()` で始まり、TTFT は `content_block_start` イベント到達時点との差で算出している。

-----

### Tool use は round-trip コストである

:::message
**原則**: ツール呼び出しは「Request → tool_use → ローカル実行 → result → 2 回目の Request → Response」という往復構造を持つ。TTFT も TTC もモデル単体の時間に往復分が加算される。
:::

:::message alert
**アンチパターン**: エージェントの「なんとなく遅い」を、モデル選択や max_tokens で解決しようとする。実際の支配因子は **ツール呼び出し回数** である。
:::

#### **ハンズオンでの具体例**

Calculator tool を使う/使わない比較で、TTFT は `1437ms → 2339ms` と約 +900ms 増えた。

| 条件         | TTFT (avg) | TTC (avg) |
| ------------ | ---------- | --------- |
| Without tool | 1437ms     | 3250ms    |
| With tool    | 2339ms     | 3910ms    |

ツールを増やすほどラウンドトリップが線形に積み上がるため、エージェント設計では「ツール数 × 平均呼び出し回数」をレイテンシ予算に含める。

-----

### Prompt caching は「同じ prefix を 2 回以上」のとき効く

:::message
**原則**: 入力単価を 1.0× とした倍率で覚える。
:::

| 操作                       | 単価 (Sonnet 4.5) | ベース倍率 |
| -------------------------- | ----------------- | ---------- |
| 通常入力                   | $3.00 / 1M        | 1.0×       |
| キャッシュ書込（5min TTL） | $3.75 / 1M        | 1.25×      |
| キャッシュ書込（1h TTL）   | $6.00 / 1M        | 2.0×       |
| キャッシュ読出             | $0.30 / 1M        | 0.1×       |
| 出力                       | $15.00 / 1M       | (変動なし) |

損益分岐:

- **5min cache**: 1.25× + 0.1× × 2 = 1.45× < 2.0×。**2 回目の読み出しで黒字**。
- **1h cache**: 2.0× + 0.1× × 3 = 2.3× < 3.0×。**3 回目の読み出しで黒字**。

:::message alert
**アンチパターン**: 1 回しか叩かないプロンプトに `cache_control` を付ける。1.25× を払うだけで読み出しに到達しないため、純粋に損。
:::

#### **ハンズオンでの具体例**

同じシステムプロンプト・同じツール定義で 2 回以上呼ぶ見込みがあるなら、5min cache は基本入れ得。マルチターン会話の system + 履歴、エージェントの system + tools 定義は典型的な caching 対象。

-----

### caching を壊す典型ミス

:::message
**原則**: caching は厳密な前方一致。プレフィックスに動的値を 1 文字でも混入させると、全リクエストが cache miss する。
:::

:::message alert
**アンチパターン**:

1. **タイムスタンプ混入**: `f"Current time: {datetime.now()}..."` を system prompt に書く。毎リクエスト 1 文字以上違うため、永遠に miss する。
2. **最低トークン未満**: Sonnet / Opus は 1,024 トークン以上、Haiku は 4,096 トークン以上のブロックでないとキャッシュされない。未満で `cache_control` を付けても **エラーは出ず黙って無視される**（silent ignore）。
3. **複数 breakpoint の放置**: マルチターンで古いターンの `cache_control` を残したまま新しいターンにも付ける。breakpoint は積み上げるのではなく、最新だけに移動する。
:::

#### **ハンズオンでの具体例**

動的な値（時刻・ユーザ ID・乱数）は user message の末尾に寄せる。**system / tools / 履歴は静的に保つ**。

-----

### caching の効きを `usage` で検証する

:::message
**原則**: caching は「効いているはず」で済ませない。レスポンスの `usage` フィールドを必ず観測する。
:::

| フィールド                          | 意味                                   |
| ----------------------------------- | -------------------------------------- |
| `usage.cache_creation_input_tokens` | 書き込んだトークン数（初回 > 0）       |
| `usage.cache_read_input_tokens`     | 読み出したトークン数（2 回目以降 > 0） |
| `usage.input_tokens`                | キャッシュ対象外の純粋な入力トークン   |

:::message alert
**アンチパターン**: ログに `usage` を残さない。誰かが system prompt にタイムスタンプを追加した瞬間に気づけず、請求額が静かに数倍化する。
:::

#### **ハンズオンでの具体例**

CI で「同一プロンプトを 2 回叩いて 2 回目の `cache_read_input_tokens > 0` をアサート」するテストを置く。これだけで構造の崩れが即座に検知できる。

-----

### Bedrock 統合は差分を把握する

:::message
**原則**: AWS Bedrock 経由でも `cache_control` のペイロード構造は同じ。ただし `usage` のフィールド名と TTL サポート範囲に差分がある。
:::

| 項目               | Anthropic 直 API                       | AWS Bedrock                                  |
| ------------------ | -------------------------------------- | -------------------------------------------- |
| クライアント       | `anthropic.Anthropic(api_key=...)`     | `anthropic.AnthropicBedrock(aws_region=...)` |
| モデル ID          | `claude-sonnet-4-5-20250929`           | `anthropic.claude-sonnet-4-5-20250929-v1:0`  |
| `usage` フィールド | snake_case (`cache_read_input_tokens`) | camelCase (`cacheReadInputTokens`)           |
| 1h TTL             | Claude 4.5+ で対応                     | Bedrock かつ Claude 4.5+ のみ                |
| スコープ           | 組織（API キー）単位                   | アカウント + リージョン + モデル ID 単位     |

:::message alert
**アンチパターン**: ログ集約・ダッシュボードを snake_case 前提で組む。Bedrock 経由のリクエストでフィールドが拾えず、命中率が見えなくなる。
:::

#### **ハンズオンでの具体例**

ログ層で両方のキーを正規化するか、両方を出力する設計にする。

-----

### Strands Agents SDK の cache 自動配置

:::message
**原則**: Strands Agents SDK（Bedrock）では `CacheConfig(strategy="auto")` を使うと、SDK が system / tools / 履歴の cache point を自動配置する。手書きの `cache_control` は不要。
:::

:::message alert
**アンチパターン**: 旧 API の `cache_tools="default"` を使い続ける。これは deprecation 方向（GitHub Issue #1577）。新規実装で採用すると将来の移行コストを抱える。
:::

#### **具体例**


```python
from strands.models.bedrock import CacheConfig

model = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20250929-v1:0",
    region_name="us-west-2",
    cache_config=CacheConfig(strategy="auto"),
)
```

-----

### Tokenizer 変更は cost-neutral ではない

:::message
**原則**: Opus 4.7 では新しい Tokenizer が入っており、同じテキストでも最大 +30% 程度トークン数が増えるケースがある。単価が下がっても、トークン数増がそれを食いつぶす可能性がある。
:::

:::message alert
**アンチパターン**: 「新モデルは単価が下がったから自動的に安い」と判断して移行する。1M context が入るからキャッシュは要らない、と判断する。1M 入ることと、1M を毎回 prefill することは別の話。長い prefix を持つほど、caching の旨味は増す。
:::

#### **ハンズオンでの具体例**

モデル更新前後で `usage.input_tokens` / `usage.output_tokens` の合計を自プロンプトで並べて比較する。コスト × トークン数の積で初めて移行判断ができる。

---

## 押さえておきたいコード/設定

### Streaming で TTFT を計測する

`client.messages.create()` は最終応答が揃ってから返るため、TTFT が測れない。必ず `client.messages.stream()` を使い、`content_block_start` イベントが立った瞬間を TTFT として記録する。

:::details TTFT measurement
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
:::

### `cache_control` を system block に打つ

system prompt は通常 string で渡すが、キャッシュを使う場合は content block の list に変える。1,024 トークン以上であること、効いていることを `usage` で確認するところまでがセット。

:::details Cache Check
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

2 回目以降で `cache_read_input_tokens` が 0 のままなら、プレフィックスのどこかが揺れているか、最低トークン数に届いていないか、のいずれか。
:::

### マルチターン会話での breakpoint 切替

マルチターンでキャッシュを効かせるコツは、**直前の assistant ターンだけに `cache_control` を残し、それより古いターンの cache_control は平文に戻す**こと。breakpoint は積み上げず、最新だけに移す。

:::details breakpoint update
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

実測（5 ターン会話）では、ターンが進むごとに `cache_read` が積み上がっていく。

```
Turn 1: 4927ms | cached: 0    | created: 1241   ← Cold (system prompt 書込)
Turn 2: 5127ms | cached: 1241 | created: 319    ← system 再利用 + Turn1 ペア書込
Turn 3: 5156ms | cached: 1560 | created: 314    ← Turn1+2 まで再利用
Turn 4: 4615ms | cached: 1874 | created: 316
Turn 5: 6498ms | cached: 2190 | created: 316
```
:::

### Bedrock 経由のクライアント生成

クライアント生成とモデル ID を差し替えるだけで、ペイロード形状は同じ。1h TTL は Bedrock + Claude 4.5+ で利用可能。

::: Bedrock
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
:::

---

## よくある勘違いと気づき

- 勘違い：「平均で速い」なら「速い」と言ってよい
  >「あなたのクライアントは平均では生きていない」の一言で、ログを集計した`avg(latency)` を何の疑問もなく週次報告に貼って緑色のままにしていた自分の愚かさが露呈した。SLA を語る場では平均ではなく p95 / p99 で語るべきだった。

- 勘違い：OTPS は `output_tokens / TTC` で計算してよい
  > `output_tokens / (TTC − TTFT)` の式が強調されるのを聞いて、`TTC` で割っていた以前の解析では「実効スループット」と「decode 速度」を取り違えていたことに気づいた。同じ指標名で違う意味を測っている、というのはKPIの誤計測の温床である。

- 勘違い：1M context があるからキャッシュは要らない
  > 1M 入ることと 1M を毎回 prefill することは別の話。「長い prefix を持つほど caching の旨味は増す」という方が正しい。

- 勘違い：system prompt にタイムスタンプを入れても問題ない
  > 何の気なしに `f"Current time: {datetime.now()}..."` を system prompt に入れていた既存コードが、永遠に cache miss していた可能性に気づいた。Cacheに持たせる情報は **静的** であるべし。

- 勘違い：`cache_control` を付ければ必ず効く
  > `cache_control` を 1,024 トークン未満のブロックに付けても **エラーが出ずに黙って無視される** という挙動は予想外だった。「効いているはず」と信じ続けて 「`cache_creation_input_tokens` が 0 のままなのに気づかない」という不具合が容易に想像できた。

---

## 現場で実践したいこと

- **SLA 議論の場で「平均」と言わない**
  - p95 / p99 で語る。`mean()` だけ書いてあるダッシュボードは、まず分布が取れる形に作り直す。Notebook の `BenchmarkResult` のように全試行を残す設計を本番計測にも持ち込む。

- **キャッシュ検証は `usage` で必ず観測する**
  - 「効いてるはず」で済ませない。CI で同一プロンプトを 2 回叩いて 2 回目の `cache_read_input_tokens > 0` をアサートするテストを入れておく。誰かが system prompt にタイムスタンプを足した瞬間に気づける。

- **Bedrock / Strands の差分にログ層で対応する**
  - Strands なら `CacheConfig(strategy="auto")` で SDK 任せにでき、旧 API の `cache_tools="default"` は使わない（Issue #1577）。Bedrock 経由では `usage` のフィールド名が `camelCase` になるため、ログ集約とダッシュボードは両方に対応させる。

---

## さらなる理解のために

セッションで紹介された一次資料のうち、特に手元に置いておきたいもの。

- **Prompt Caching**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- **Streaming Messages**: https://docs.anthropic.com/en/api/messages-streaming
- **Token Counting API**: https://docs.anthropic.com/en/api/messages-count-tokens
- **Message Batches API**: https://docs.anthropic.com/en/docs/build-with-claude/batch-processing — オフライン処理 50% 引き
- **AWS Bedrock prompt caching**: https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
- **Strands Agents SDK**: https://strandsagents.com/
- **OpenTelemetry GenAI semantic conventions**: https://opentelemetry.io/docs/specs/semconv/gen-ai/ — TTFT / TTC を分散トレースに乗せるための標準

---

## 章末 — 工学責任と次章へ

第 1 章で「IT の役割は変わる」という takeaway を書いた。「コードを書く人から、判断の輪郭を引く人へ」。

この章を経て、その輪郭にもう一本線が足された気がしている。クライアントとの議論の場で「動きます」と言う代わりに、「p99 でこれを満たします」「キャッシュ命中率がこの値を割ったらアラートを上げます」と言える人になる。動くかどうかではなく、どの確率で・どの遅さで・いくらで動くか。それを数字で約束できる立場を、自分から取りに行く。

これが、今回学んだ「p99 で語る」という言葉の、もう一つの意味だと思う。

次章では、推論最適化と表裏一体の **Context Engineering** に踏み込む。速く・安く回せるようになった土台の上に、どのような情報を・どの順序で・どの形式で渡せばモデルが間違えないか。`Context Engineering` の Notebook を通して、`day2/03_context-engineering/` の体験を整理する。

→ 次章: [08-context-engineering](./08-context-engineering.md)
