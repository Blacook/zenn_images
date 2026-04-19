---
title: "AI時代の設計指針：AIDDとOOPのアウフヘーベン「CUPID特性」のすゝめ"
emoji: "💘"
type: "idea"
topics: ["AIDD", "OOP", "architecture", "SOLID", "CUPID"]
published: false
---

## はじめに

日々の開発でAIコーディングエージェント（Claude CodeやCursorなど）をフル活用されている方も多いのではないでしょうか。

AIが爆速でコードを書いてくれる現代、我々開発者の悩みは「どう書くか」から「**生成された大量のコードをどう管理し、品質を保つか**」へと移り変わっています。そこで今、改めて注目したいのが、従来の「SOLID原則」に代わる新しい指針 **「CUPID特性」** です。

今回は、AI駆動開発（AIDD）の勢いと、オブジェクト指向（OOP）の秩序を高い次元で統合（アウフヘーベン）するための考え方として、このCUPID特性を紐解いていきたいと思います。

---

## 1. SOLIDの「辛さ」— "良い原則"が窮屈になる場面

最初に断っておきますが、**SOLIDは今でも優れた設計の羅針盤**です。「OCP（開放閉鎖原則）のおかげでバグを最小化できた」「SRP（単一責任原則）がコードレビューの共通言語になった」という経験をお持ちの方も多いでしょう。

問題は原則そのものではありません。**AI駆動開発（AIDD）が当たり前になった現代において、SOLIDが前提としていた「人間のペース」と「手書きのコード量」が根本から変わってしまった**ことにあります。

### SOLIDの「辛さ」まとめ

| 原則            | 良い意図     | AI時代に生じる辛さ                                         |
| :-------------- | :----------- | :--------------------------------------------------------- |
| SRP（単一責任） | 責任の分離   | クラスが爆発し、処理の流れがファイルをまたいで追えない     |
| OCP（開放閉鎖） | 拡張への開放 | 抽象層が深くなり、AIが修正箇所を特定できない               |
| LSP（置換可能） | 代替可能性   | 継承階層の「意味的な契約違反」が実行時まで発覚しない       |
| ISP（IF分離）   | 依存の最小化 | インターフェースが細分化されすぎ、AI指示の文脈コストが増大 |
| DIP（依存逆転） | 依存の逆転   | DIコンテナが肥大化し、AIへの文脈共有が困難になる           |

### 「過適用」が起きる構造的な理由

SOLIDは、いずれの原則も「準拠しているか / 違反しているか」という**二値判断を促す設計**になっています。その結果、善意の開発者ほど「完全に準拠しなければ」と過適用に走りやすい。AIが一気に生成した数百行のコードを前にすると、この傾向はさらに加速します。

以下に、各原則が「辛さ」に転じる典型パターンをPythonコードで示します。

---

#### SRP（単一責任原則）の辛さ — クラスが爆発する

「メール送信処理を直したい」と思ったとき、どのクラスを開けばいいか、AIに聞かないと分からない——そんな状況に陥りがちです。

```python
# ❌ SRPを厳格に適用した結果、処理の流れが5つのクラスに分散
class UserFetcher:
    def fetch(self, user_id: int) -> dict: ...

class UserValidator:
    def validate(self, user: dict) -> bool: ...

class EmailFormatter:
    def format(self, user: dict) -> str: ...

class EmailSender:
    def send(self, email: str) -> None: ...

class UserNotificationOrchestrator:
    # 「変更の理由は1つ」を守ったら、処理の流れを追うために4つのファイルを開く必要が生じた
    def __init__(self, fetcher, validator, formatter, sender): ...
    def notify(self, user_id: int) -> None: ...
```

---

#### OCP（開放閉鎖原則）の辛さ — 抽象層が深くなる

AIが「このクラスの振る舞いを変更したい」と指示されても、継承の上流まで文脈を遡れず、局所的な修正に終始してしまいます。

```python
# ❌ 「拡張に開く」を徹底した結果、抽象クラスが3層になった
from abc import ABC, abstractmethod

class BaseNotifier(ABC):
    @abstractmethod
    def notify(self, message: str) -> None: ...

class BaseAsyncNotifier(BaseNotifier, ABC):
    @abstractmethod
    async def notify_async(self, message: str) -> None: ...

class BaseRetryableAsyncNotifier(BaseAsyncNotifier, ABC):
    @abstractmethod
    def get_retry_policy(self) -> dict: ...

# 実装クラスに辿り着くまでに3ファイルを遡る必要がある
class EmailNotifier(BaseRetryableAsyncNotifier):
    ...
```

---

#### LSP（リスコフの置換原則）の辛さ — 型エラーが連鎖する

「このクラスを使い回せるはずなのに、実行時に初めて例外が発覚する」——LSP違反は静的解析をすり抜け、AIレビューでも見落とされやすいのが厄介なところです。

```python
# ❌ 継承で「is-a」関係を表現しようとした結果、
#    サブクラスが親の契約を静かに破壊している

class Notification:
    def send(self, user_id: int, message: str) -> bool:
        """送信成功/失敗をboolで返す"""
        ...

class BulkNotification(Notification):
    def send(self, user_id: int, message: str) -> bool:
        # ❌ 大量送信時はレート制限で例外を投げることがある
        #    親クラスの「必ずboolを返す」契約を破っている
        if self._is_rate_limited():
            raise RateLimitError("送信レート超過")  # 親の呼び出し元は例外を想定していない
        ...

class SilentNotification(Notification):
    def send(self, user_id: int, message: str) -> bool:
        # ❌ 「送信しない」仕様なのにTrueを返す
        #    LSP準拠に見せかけているが、意味的な契約は破綻している
        return True
```

```python
# 呼び出し側は「どのNotificationでも同じように使える」と信じているが...
def notify_all(notifier: Notification, users: list[int]) -> None:
    for user_id in users:
        # BulkNotificationを渡されると突然例外が飛んでくる
        # AIがこのバグを発見するには、継承ツリー全体を把握する必要がある
        notifier.send(user_id, "お知らせ")
```

---

#### ISP（インターフェース分離原則）の辛さ — 細分化しすぎて管理コストが増大

「依存を最小化する」という意図は正しい。しかし細分化が行き過ぎると、**「どこに何を足せばいいか」の判断コストが、インターフェース数に比例して増大**します。AIへの指示文に「どのインターフェースを修正すべきか」を都度説明する羽目になります。

```python
# ❌ ISPを厳格に適用した結果、インターフェースが9つに分裂した

from abc import ABC, abstractmethod

class IEmailSendable(ABC):
    @abstractmethod
    def send_email(self, to: str, body: str) -> None: ...

class IEmailTemplatable(ABC):
    @abstractmethod
    def render_template(self, template_id: str, context: dict) -> str: ...

class IEmailLoggable(ABC):
    @abstractmethod
    def log_sent(self, to: str, status: str) -> None: ...

class IEmailRetryable(ABC):
    @abstractmethod
    def retry(self, to: str, body: str, attempts: int) -> None: ...

class IEmailQueuable(ABC):
    @abstractmethod
    def enqueue(self, to: str, body: str) -> None: ...

# ... さらにIEmailSchedulable, IEmailTrackable, IEmailBatchable, IEmailCancellableが続く

# 実装クラスは全部を継承することを強いられる
class FullEmailService(
    IEmailSendable,
    IEmailTemplatable,
    IEmailLoggable,
    IEmailRetryable,
    IEmailQueuable,
    # ...
):
    # AIに「メール送信機能を拡張して」と頼んだとき、
    # どのインターフェースに新メソッドを追加すべきか判断できない
    ...
```

---

#### DIP（依存性逆転原則）の辛さ — 初期設定地獄

AIに「メール送信ロジックを変えて」と頼む際、変更すべき実装クラスだけでなく「どのregistrationに対応するのか」という文脈も一緒に渡す必要があります。設定ファイルが肥大化すると、AIが局所的な変更指示を正確に解釈するために必要な文脈量が膨れ上がり、的外れな修正を生成しやすくなります。

```python
# ❌ DIPの正しい適用だが、DIコンテナの設定ファイルが100行を超えた
# container.py
container.register(IUserRepository, SqlUserRepository)
container.register(IEmailService, SmtpEmailService)
container.register(INotificationQueue, SqsNotificationQueue)
container.register(IRetryPolicy, ExponentialBackoffPolicy)
# ... 以下延々と続く

# AIに「メール送信ロジックを変えて」と頼んでも、
# どのregistrationを触ればいいか文脈として渡しきれない
```

---

## 2. CUPID特性 — 厳格な原則に代替する設計特性

SOLIDが「守らなければならないルール」だとすれば、CUPIDは「**コードが自然と近づいていくべき方向性**」です。合否の判定ではなく、グラデーションで語れることがAI時代との相性の良さに直結します。

### CUPIDの5つの要素

| 特性                | 意味           | 大切なポイント                           | 引き継ぐSOLIDの辛さ | CUPIDが加える視点          | AIへの指示軸                       |
| :------------------ | :------------- | :--------------------------------------- | :------------------ | :------------------------- | :--------------------------------- |
| **C**omposable      | 組み合わせ可能 | どこでも楽に使える、依存が少ない         | SRP・ISPの細分化    | 外から見た使いやすさ       | 「他の場所で使い回せるか」         |
| **U**nix Philosophy | Unix哲学       | 一つのことを、見事に成し遂げる           | SRPの内部管理コスト | 目的を一文で言い切れるか   | 「動詞＋名詞で表現できるか」       |
| **P**redictable     | 予測可能       | 驚きがなく、意図通りに動く               | LSPの代替可能性     | 驚き最小・副作用の明示     | 「名前と振る舞いが一致しているか」 |
| **I**diomatic       | 慣習的         | その言語やチームの「らしさ」を大切にする | （SOLIDにない概念） | 言語・チームの慣習への共感 | 「プロジェクト慣習リストを渡す」   |
| **D**omain-based    | ドメインベース | 技術用語ではなく、ビジネスの言葉で語る   | DIPの依存整理       | ビジネス語彙での命名       | 「ドメイン語彙一覧を渡す」         |

---

### C — Composable（組み合わせ可能）

#### 意図

「このモジュール、他の場所でも使いたいな」と思ったとき、すんなり使い回せること。依存が少なく、入出力が明確で、文脈に依存しない単位で設計されていること。

#### SOLIDとの関係

SRP・ISPの**「辛さ」を引き継ぎ、外側の視点で再定義**します。SRPが「内部の変更理由を1つにせよ」と問うのに対し、Composableは「**外から見て気持ちよく使えるか**」を問います。インターフェースを細分化するのではなく、「この関数/クラスを別の場所にコピーしたとき、ちゃんと動くか」が判断基準です。

#### Claude Codeプロンプト例

```
このEmailNotifierクラスをComposableな設計に直してください。
基準は「このクラスをコピーして別のサービスで使い回せるか」です。
外部サービスへの直接依存があれば引数や設定として注入する形に変えてください。
```

#### Python Before → After

```python
# ❌ Before: SmtpサーバとDBに直接依存しており、他では使い回せない
import smtplib
import psycopg2

class EmailNotifier:
    def notify(self, user_id: int, message: str) -> None:
        # DBから直接ユーザー情報を取得
        conn = psycopg2.connect("postgresql://...")
        user = conn.execute(f"SELECT email FROM users WHERE id={user_id}").fetchone()

        # SMTPサーバに直接接続
        with smtplib.SMTP("smtp.example.com", 587) as smtp:
            smtp.sendmail("no-reply@example.com", user["email"], message)
```

```python
# ✅ After: 依存を外から注入。どこでも使い回せる
from dataclasses import dataclass
from typing import Protocol

class UserRepository(Protocol):
    def find_email(self, user_id: int) -> str: ...

class MailClient(Protocol):
    def send(self, to: str, body: str) -> None: ...

@dataclass
class EmailNotifier:
    """ユーザーへのメール通知。依存はすべて外から注入される"""
    user_repository: UserRepository
    mail_client: MailClient

    def notify(self, user_id: int, message: str) -> None:
        email = self.user_repository.find_email(user_id)
        self.mail_client.send(to=email, body=message)
```

`UserRepository`と`MailClient`をProtocolで抽象化することで、本番ではSMTP、テストではインメモリの偽実装を差し込めます。AIに「このクラスのテストを書いて」と頼んだときも、依存が明示されているため迷わず正しいモックを生成してくれます。

---

### U — Unix Philosophy（Unix哲学）

#### 意図

「一つのことを、見事に成し遂げる」。モジュールの目的を**動詞+名詞の一文**で言い切れること。「〇〇して、さらに△△もして、必要なら□□も」と複数の目的が混在していないこと。

#### SOLIDとの関係

SRPの**「内側」から「外側」へのパラダイムシフト**です。SRPは「変更の理由が1つであること」を問いますが、それはコードの内部構造を人間が細かく管理することを前提にしています。Unix Philosophyは「**このモジュールが何をするか、一言で言えるか**」を問います。AIへの指示文と相性が良く、「CSVを読んで集計する関数」ではなく「CSVを読む関数」と「集計する関数」に分けることで、AIが各処理の文脈を正確に把握できます。

#### Claude Codeプロンプト例

```
この関数はCSVの読み込み・バリデーション・集計・レポート出力を
1つの関数でやっています。
Unix Philosophyに従い「一つのことだけを見事にやる」単位に分割してください。
各関数の目的は「動詞+名詞」の一文で表現できるように命名してください。
```

#### Python Before → After

```python
# ❌ Before: 「なんでもやる」関数。修正箇所を特定するだけで一苦労
def process_sales_report(filepath: str) -> None:
    import csv, smtplib
    rows, errors = [], []

    with open(filepath) as f:
        reader = csv.DictReader(f)
        for row in reader:
            # バリデーション
            if not row.get("amount"):
                errors.append(row)
                continue
            # 集計
            rows.append({"region": row["region"], "amount": float(row["amount"])})

    # 集計
    totals = {}
    for row in rows:
        totals[row["region"]] = totals.get(row["region"], 0) + row["amount"]

    # メール送信
    body = "\n".join(f"{r}: {a}" for r, a in totals.items())
    with smtplib.SMTP("smtp.example.com") as smtp:
        smtp.sendmail("report@example.com", "manager@example.com", body)
```

```python
# ✅ After: 各関数が「一つのことだけ」を担う
import csv
from dataclasses import dataclass

@dataclass
class SalesRecord:
    region: str
    amount: float

# 「CSVを読む」だけ
def load_sales_csv(filepath: str) -> tuple[list[SalesRecord], list[dict]]:
    """CSVを読み込み、有効レコードとエラー行を返す"""
    records, errors = [], []
    with open(filepath) as f:
        for row in csv.DictReader(f):
            if not row.get("amount"):
                errors.append(row)
            else:
                records.append(SalesRecord(row["region"], float(row["amount"])))
    return records, errors

# 「地域別に集計する」だけ
def aggregate_by_region(records: list[SalesRecord]) -> dict[str, float]:
    """地域ごとの売上合計を返す"""
    totals: dict[str, float] = {}
    for record in records:
        totals[record.region] = totals.get(record.region, 0) + record.amount
    return totals

# 「レポート文字列を生成する」だけ
def format_sales_report(totals: dict[str, float]) -> str:
    """集計結果を人間が読めるレポート文字列に変換する"""
    return "\n".join(f"{region}: {amount:,.0f}円" for region, amount in totals.items())
```

「集計ロジックだけ直して」とAIに頼んだとき、`aggregate_by_region`だけを渡せばよく、他の処理が巻き込まれる心配がありません。

---

### P — Predictable（予測可能）

#### 意図

「名前を見ただけで何をするか分かる」「同じ入力を与えたら必ず同じ結果が返る」「副作用がどこで起きるか明示されている」。コードを読んだ人間もAIも、**驚かない**こと。

#### SOLIDとの関係

LSPの「代替可能性」を**「驚き最小の原則」として再解釈**します。LSPは型システムや継承契約の観点から代替可能性を語りますが、Predictableはもっと広く「このコードの振る舞いは名前と一致しているか」を問います。副作用の隠蔽・暗黙の状態変化・文脈依存の戻り値、これらすべてが「予測不能」の原因であり、AIレビューで見落とされやすい箇所でもあります。

#### Claude Codeプロンプト例

```
このコードをPredictableな設計にレビューしてください。
チェック観点は以下の3点です:
1. 関数名と実際の振る舞いが一致しているか
2. 同じ引数で必ず同じ結果を返すか（副作用・グローバル状態への依存がないか）
3. 例外やNoneを返す条件が呼び出し側から予測できるか
問題箇所を指摘し、修正案を提示してください。
```

#### Python Before → After

```python
# ❌ Before: 名前と振る舞いが一致していない「驚き」が3か所ある
import random

_sent_count = 0  # グローバル状態

def get_user_email(user_id: int) -> str | None:
    # ❌ 驚き①: 「取得する」関数なのに送信ログをDBに書き込む副作用がある
    _log_access(user_id)

    user = _fetch_from_db(user_id)
    if not user:
        return None

    # ❌ 驚き②: 同じuser_idでも確率でNoneを返す（テスト不可能）
    if random.random() < 0.01:
        return None

    return user["email"]

def send_notification(email: str, message: str) -> bool:
    global _sent_count
    # ❌ 驚き③: グローバル変数を書き換える副作用が関数名から読み取れない
    _sent_count += 1
    return _actually_send(email, message)
```

```python
# ✅ After: 名前・副作用・戻り値がすべて予測可能
from dataclasses import dataclass
from typing import Optional

@dataclass(frozen=True)
class NotificationResult:
    success: bool
    sent_count: int  # 副作用の結果を戻り値として明示

# 「取得する」だけ。副作用なし・決定的
def find_user_email(user_id: int) -> Optional[str]:
    """DBからメールアドレスを取得する。存在しない場合はNoneを返す"""
    user = _fetch_from_db(user_id)
    return user["email"] if user else None

# 副作用（送信）を担うが、結果を戻り値で明示
def send_notification(email: str, message: str, current_count: int) -> NotificationResult:
    """通知メールを送信し、更新後の送信カウントを返す"""
    success = _actually_send(email, message)
    return NotificationResult(
        success=success,
        sent_count=current_count + 1 if success else current_count
    )
```

「この関数は何をするか」がコードを読む前から予測できることで、AIが生成したコードのレビューコストが大幅に下がります。

---

### I — Idiomatic（慣習的）

#### 意図

「このコードはPythonらしいか」「このプロジェクトの書き方と揃っているか」。言語仕様の慣習だけでなく、**チーム・プロジェクト固有のコーディングスタイルや語彙**にも従っていること。次にそのコードを読む人間への共感が根底にある。

#### SOLIDとの関係

SOLIDには**「慣習」や「文化」を扱う原則が存在しません**。これはIdiomaticがCUPIDで初めて明示した概念です。AIはPythonでもJavaのようなgetterを量産したり、プロジェクト固有の命名規則を無視したコードを生成することがあります。Idiomaticはそれを「修正すべき状態」と定義し、AIへの指示軸を与えます。

#### Claude Codeプロンプト例

```
このコードをIdiomaticなPythonに修正してください。
以下のプロジェクト慣習に従ってください:
- dataclassではなくPydanticのBaseModelを使う
- getter/setterは使わずプロパティ直接アクセス
- 戻り値の型ヒントを必ず明示する
- 変数名はsnake_case、クラス名はPascalCase
- 日本語のdocstringを必ず書く
```

#### Python Before → After

```python
# ❌ Before: Pythonなのに「Javaらしい」書き方が混在している

class NotificationConfig:
    def __init__(self):
        self.__smtp_host = ""      # ❌ Javaスタイルのprivateフィールド
        self.__smtp_port = 0
        self.__max_retry = 3

    # ❌ getterを律儀に定義している（Javaの慣習）
    def getSmtpHost(self) -> str:  # ❌ camelCaseのメソッド名
        return self.__smtp_host

    def setSmtpHost(self, host: str) -> None:
        self.__smtp_host = host

    def getMaxRetry(self) -> int:
        return self.__max_retry

    def setMaxRetry(self, retry: int) -> None:
        if retry < 0:
            raise ValueError("retry must be non-negative")
        self.__max_retry = retry

class notificationService:   # ❌ クラス名がsnake_case
    def SendNotification(self, userId, msg):  # ❌ 型ヒントなし、camelCase
        config = NotificationConfig()
        config.setSmtpHost("smtp.example.com")
        config.setMaxRetry(5)
        # 処理...
```

```python
# ✅ After: Pythonらしく、プロジェクト慣習に沿った書き方
from pydantic import BaseModel, Field, field_validator

class NotificationConfig(BaseModel):
    """通知サービスの接続設定"""
    smtp_host: str
    smtp_port: int = 587
    max_retry: int = Field(default=3, ge=0)  # ge=0でバリデーションを宣言的に表現

    @field_validator("max_retry")
    @classmethod
    def validate_max_retry(cls, v: int) -> int:
        """リトライ回数は0以上でなければならない"""
        return v  # Fieldのge=0が前段で保証するため、ここは通過したものを返すだけ

class NotificationService:
    """ユーザーへのメール通知を担うサービス"""

    def __init__(self, config: NotificationConfig) -> None:
        self._config = config

    def send_notification(self, user_id: int, message: str) -> bool:
        """指定ユーザーに通知メールを送信する。成功時はTrueを返す"""
        # 処理...
        return True
```

> AIに「このプロジェクトの慣習」をプロンプトで明示するだけで、生成コードの品質が大きく変わります。SKILL.mdやCLAUDE.mdにプロジェクト固有の慣習を書き込んでおくと、毎回説明する手間が省けます。

---

### D — Domain-based（ドメインベース）

#### 意図

コードの語彙が、技術用語ではなくビジネスの言葉で語られていること。`Manager`・`Handler`・`Processor`のような**技術的な汎用語を避け**、「注文」「通知」「承認」など**ドメインの言葉**でモジュールや関数を命名すること。

#### SOLIDとの関係

DIPの「依存性逆転」は技術レイヤーの依存方向を整理しますが、**語彙や命名には踏み込みません**。Domain-basedはその一歩先として、「技術的に正しい抽象化」と「ビジネス担当者にも読める語彙」を両立させます。DDDの「ユビキタス言語」に近い概念ですが、CUPIDはより軽量に「命名の方向性」として提示しています。

AIは文脈を理解してコードを生成しますが、命名に迷うと`DataManager`や`ServiceHelper`のような**意味のない汎用名**を多用しがちです。ドメイン語彙を明示することで、AIが生成するコードの可読性が一段上がります。

#### Claude Codeプロンプト例

```
このコードをDomain-basedな命名に直してください。
このシステムのドメイン語彙は以下の通りです:
- 「通知」= Notification（メール・Slackを問わない概念）
- 「配信」= Delivery（実際の送信処理）
- 「受信者」= Recipient（userやcustomerではなく）
- 「配信チャネル」= DeliveryChannel（smtp/slack/webhookの総称）
Manager, Handler, Processor, Helperという命名は使わないでください。
```

#### Python Before → After

```python
# ❌ Before: 技術用語と汎用語が混在し、何のシステムか読み取れない

class DataManager:           # ❌ 何のデータ？
    def process(self, data): # ❌ 何を処理する？
        ...

class EmailHandler:          # ❌ 「処理する」だけで目的が不明
    def handle(self, payload: dict) -> bool:
        user_id = payload["user_id"]   # ❌ userなのかrecipientなのか
        msg = payload["msg"]           # ❌ msgはmessageか、bodyか
        return self._send(user_id, msg)

class NotificationProcessor:  # ❌ processorは何をprocessするのか
    def __init__(self):
        self.handler = EmailHandler()

    def run(self, raw_data: list[dict]) -> None:  # ❌ runは何を実行する？
        for item in raw_data:
            self.handler.handle(item)
```

```python
# ✅ After: ドメインの語彙でコードが「仕様書」として読める
from dataclasses import dataclass
from typing import Protocol

@dataclass(frozen=True)
class Notification:
    """配信すべき通知の内容"""
    recipient_id: int
    body: str

@dataclass(frozen=True)
class DeliveryResult:
    """通知の配信結果"""
    notification: Notification
    success: bool
    channel: str

class DeliveryChannel(Protocol):
    """配信チャネルの共通インターフェース（メール・Slack・Webhookなど）"""
    def deliver(self, notification: Notification) -> DeliveryResult: ...

class NotificationDispatcher:
    """通知を受け取り、指定チャネルで配信するディスパッチャー"""

    def __init__(self, channel: DeliveryChannel) -> None:
        self._channel = channel

    def dispatch_all(self, notifications: list[Notification]) -> list[DeliveryResult]:
        """全通知を配信し、各配信結果を返す"""
        return [self._channel.deliver(n) for n in notifications]
```

ビジネス担当者に「この`NotificationDispatcher`は何をするクラスですか？」と聞いたとき、コードを読まずとも答えられる——それがDomain-basedの目指す状態です。また、AIに「新しい配信チャネルを追加して」と頼んだときも、`DeliveryChannel`プロトコルを実装すれば良いと即座に判断できます。

---

## 3. AIDDとCUPIDの連携パターン — Claude Codeでの実践

CUPIDはコードの「あるべき方向性」を示しますが、その真価はAIツールと組み合わせたときに発揮されます。ここでは**Claude Codeを使った開発サイクル**を「生成 → レビュー → リファクタリング」の3フェーズに分け、各フェーズでCUPIDをどう活かすかを具体的に示します。

### 実戦のポイント

| フェーズ             | やること                        | 活用するCUPID特性       | Claude Codeでのポイント                           |
| :------------------- | :------------------------------ | :---------------------- | :------------------------------------------------ |
| **生成**             | CUPID制約をプロンプトに埋め込む | 全5特性                 | CLAUDE.mdにIdiomatic・Domain語彙を常駐させる      |
| **レビュー**         | CUPIDチェックリストで査読       | 全5特性                 | 5観点の判定フォーマットを固定して比較しやすくする |
| **リファクタリング** | 観点を絞って改善依頼            | Idiomatic・Domain-based | ロジック変更と命名変更は必ず別依頼に分ける        |

---

### フェーズ1 — 生成：CUPID制約をプロンプトに埋め込む

AIへのコード生成依頼で最も効果的なのは、**「何を作るか」と同時に「どんな特性を持たせるか」を指定する**ことです。

#### 効果の薄い指示

```
メール通知機能を実装してください。
```

これだとAIは「動くコード」を返しますが、外部依存が直結していたり、命名が汎用的すぎたり、テストしにくい構造になりがちです。

#### CUPIDを埋め込んだ指示

```
以下の仕様でメール通知機能を実装してください。

【仕様】
- ユーザーIDとメッセージ本文を受け取り、該当ユーザーにメールを送信する
- 送信失敗時はリトライを最大3回行う

【CUPIDの制約】
- Composable: SMTPクライアントとユーザー取得処理はProtocolで注入する
- Unix Philosophy: 「メールを送る」「リトライする」「ユーザーを探す」を別関数に分ける
- Predictable: 副作用（実際の送信）を持つ関数は戻り値でResultを返す
- Idiomatic: Pydanticで設定管理、型ヒント必須、日本語docstring
- Domain-based: Manager/Handler/Processor禁止。通知=Notification、配信=Deliveryで統一

【言語・環境】Python 3.12、外部ライブラリはpydantic・tenacityのみ
```

制約を明示することで、AIが「設計の選択に迷う部分」を人間の意図通りに埋めてくれます。特に**Idiomatic**と**Domain-based**の制約は、プロジェクトの`CLAUDE.md`にあらかじめ書いておくと毎回の入力が不要になります。

#### CLAUDE.mdへの記載例

```markdown
## コーディング規約（CUPID準拠）

### Idiomatic

- 設定管理: Pydantic BaseModel
- 型ヒント: すべての引数・戻り値に必須
- docstring: 日本語、クラスと公開メソッドに必須
- 命名: 変数・関数はsnake_case、クラスはPascalCase

### Domain-based（通知サブシステム）

- 通知の概念: Notification
- 実際の送信行為: Delivery / deliver
- 送り先: Recipient（user/customerは使わない）
- 送信経路: DeliveryChannel
- 禁止語: Manager, Handler, Processor, Helper, Util
```

---

### フェーズ2 — レビュー：CUPIDチェックリストでAIに査読させる

AIが生成したコードを再度AIにレビューさせる「**AIレビューループ**」は、品質向上の強力な手段です。このとき「なんとなくレビューして」ではなく、**CUPIDの5観点を明示したチェックリスト**を渡すことで、指摘の精度が上がります。

#### CUPIDレビュープロンプトテンプレート

```
以下のコードをCUPIDの5観点でレビューしてください。
各観点で「問題あり / 問題なし」を判定し、
問題ありの場合は該当箇所と改善案を具体的に示してください。

【レビュー観点】
C - Composable: 外部依存が直結していないか。他のコンテキストで使い回せるか。
U - Unix Philosophy: 1つの関数/クラスが複数の目的を持っていないか。
P - Predictable: 名前と振る舞いが一致しているか。副作用が隠れていないか。
I - Idiomatic: プロジェクト規約（CLAUDE.md参照）に沿っているか。
D - Domain-based: ビジネス語彙を使っているか。汎用語（Manager等）を避けているか。

【レビュー対象コード】
{ここにコードを貼る}
```

#### レビュー結果の活用パターン

AIのレビュー結果は以下の形式で返ってくることが多いです。

```
C - Composable: ❌ 問題あり
  → EmailNotifierがsmtplibに直接依存しています。
    MailClientをProtocolに切り出し、外から注入する形を推奨します。

U - Unix Philosophy: ✅ 問題なし

P - Predictable: ❌ 問題あり
  → send_notification()がグローバル変数_sent_countを書き換えています。
    副作用を戻り値のNotificationResultに含める形に変更してください。

I - Idiomatic: ❌ 問題あり
  → getSmtpHost()のようなcamelCaseのgetterが存在します。
    Pythonらしくプロパティ直接アクセスに変更してください。

D - Domain-based: ✅ 問題なし
```

このフォーマットで返ってくると、**どの観点の修正を優先するか**を人間が判断しやすくなります。全指摘を一度に直すのではなく、C→P→Iの順に1観点ずつ修正サイクルを回すのが実践的です。

---

### フェーズ3 — リファクタリング：Idiomatic・Domain-basedを軸に改善指示を出す

既存コードの改善フェーズでは、**「全体をまとめて直して」ではなく観点を絞って依頼する**ことが重要です。特にIdiomaticとDomain-basedは、コードの構造を大きく変えずに**語彙と書き方だけを揃える**リファクタリングに適しており、AIとの相性が抜群です。

#### Idiomaticリファクタリングの指示例

```
このファイルをIdiomaticなリファクタリングのみ行ってください。
ロジックは一切変えず、以下だけ修正してください:
1. getter/setterをプロパティ直接アクセスに変える
2. camelCaseのメソッド名をsnake_caseに統一する
3. 型ヒントが欠けている引数・戻り値に追加する
4. docstringが欠けているクラス・公開メソッドに日本語で追加する

修正前後のdiffも合わせて提示してください。
```

#### Domain-basedリファクタリングの指示例

```
このファイルのクラス名・メソッド名・変数名を
Domain-basedな命名にリネームしてください。
ロジックは変えず、名前だけ変更します。

【変換ルール】
- DataManager → NotificationDispatcher
- EmailHandler → EmailDeliveryChannel
- process() → dispatch_all()
- handle() → deliver()
- user_id → recipient_id
- msg / message → notification.body

変更箇所の一覧表も出力してください（旧名 → 新名）。
```

ロジックと命名を**別の依頼として分離する**ことで、AIの修正範囲が明確になり、レビューコストも下がります。

---

## 4. SOLID → CUPID 継承マッピング — 捨てるのではなく、引き継ぐ

ここまで読んでいただいた方は、「CUPIDはSOLIDを否定しているのではなく、**SOLIDの良い部分を現代の文脈で言い換えている**」という感覚を掴んでいただけたのではないでしょうか。

この章では、その関係を「継承マッピング」として整理します。SOLIDの各原則が「何を守ろうとしていたか」と、CUPIDの各特性が「それをどう引き継ぎ、何を加えたか」を対応させています。

---

### 継承マッピング全体図

| SOLID原則       | 守っていたもの |  →  | 引き継ぐCUPID | CUPIDが加えた視点                      |
| :-------------- | :------------- | :-: | :------------ | :------------------------------------- |
| SRP（単一責任） | 変更理由の分離 |  →  | U・C          | 「目的の明確さ」「外からの使いやすさ」 |
| OCP（開放閉鎖） | 拡張への備え   |  →  | C             | Protocol差し込みによるフラットな拡張性 |
| LSP（置換可能） | 型契約の一貫性 |  →  | P             | 意味的な驚きのなさ・副作用の明示       |
| ISP（IF分離）   | 依存の最小化   |  →  | C             | 細分化せずProtocolで必要契約のみ表現   |
| DIP（依存逆転） | 抽象への依存   |  →  | D             | 抽象の命名をビジネス語彙で行う         |
| （なし）        | —              |  →  | I             | 言語・チーム固有の慣習への共感（新設） |

---

### 各継承関係の詳細

#### SRP → U（Unix Philosophy）＋ C（Composable）

SRPが「**変更の理由を1つに絞れ**」と内部構造に注目したのに対し、CUPIDは2方向に引き継ぎます。

「目的を1つに絞れ」という方向性は**Unix Philosophyへ**。「外から使い回せる単位にせよ」という方向性は**Composableへ**。SRPの意図を分解することで、「何を守るために責任を分けるのか」がより明確になります。

|        | SRP                        | U + C                                            |
| :----- | :------------------------- | :----------------------------------------------- |
| 注目点 | 内部の変更理由             | 外から見た目的・使いやすさ                       |
| 問い   | 「変更理由は1つか？」      | 「一文で目的を言えるか？」「他で使い回せるか？」 |
| AI指示 | 判断しにくい（文脈が必要） | 判断しやすい（一文で表現できるか確認するだけ）   |

#### OCP → C（Composable）

OCPが「**拡張に開き、修正に閉じよ**」と言うとき、その背景にあるのは「既存コードを壊さずに機能を足したい」という意図です。CUPIDはこれをComposableとして引き継ぎ、「**新しい実装をProtocolに差し込める構造になっているか**」という問いに変換します。

```python
# OCP的発想（継承で拡張に備える）→ 階層が深くなりがち
class BaseNotifier(ABC): ...
class BaseAsyncNotifier(BaseNotifier, ABC): ...
class EmailNotifier(BaseAsyncNotifier): ...

# Composable的発想（Protocolで差し込みに備える）→ 階層がフラット
class DeliveryChannel(Protocol):
    def deliver(self, notification: Notification) -> DeliveryResult: ...

# 新チャネルの追加はProtocolを満たすクラスを作るだけ
class SlackDeliveryChannel:
    def deliver(self, notification: Notification) -> DeliveryResult: ...
```

#### LSP → P（Predictable）

LSPが「**サブクラスは親クラスと置換可能であれ**」と言うとき、守りたかったのは「使う側が驚かないこと」です。CUPIDはこれをPredictableとして引き継ぎ、型システムの文脈だけでなく「**名前・副作用・戻り値すべてにおいて驚きがないこと**」へと拡張します。

LSPは「型の契約を守れ」という静的な観点でしたが、Predictableは「動的な振る舞いも含めて予測可能か」を問います。AIが生成したコードに潜む「静的解析を通過するが意味的に裏切る」バグを発見する観点として、より実践的です。

#### ISP → C（Composable）

ISPが「**使わないメソッドへの依存を強制するな**」と言うとき、守りたかったのは「必要なものだけに依存すること」です。CUPIDはこれをComposableとして引き継ぎ、「**インターフェースを細分化する**」のではなく「**Protocolで必要な契約だけを表現する**」方向に昇華します。細分化の管理コストを払わずに、ISPの意図を達成できます。

#### DIP → D（Domain-based）

DIPが「**具体ではなく抽象に依存せよ**」と言うとき、技術レイヤーの整理には有効でしたが、「何という名前の抽象に依存すべきか」には踏み込みませんでした。CUPIDはDomain-basedとして引き継ぎ、「**抽象の命名をビジネス語彙で行う**」ことを加えます。

`IEmailService`ではなく`DeliveryChannel`。`IUserRepository`ではなく`RecipientFinder`。技術的な正しさにビジネスの語彙を重ねることで、コードが「仕様書」として機能し始めます。

#### Idiomatic — SOLIDが扱わなかった領域

SOLIDの5原則はすべて「コードの構造」に関するものでした。「**このコードはPythonらしいか**」「**このプロジェクトの慣習に合っているか**」という問いは、SOLIDの射程外です。

AI駆動開発では「動くが文化的に異質なコード」が大量生成されるリスクが高まります。IdiomaticはそのリスクをCUPIDが新設した観点として正面から扱い、CLAUDE.mdやSKILL.mdといった「AIへの慣習の伝え方」と直接接続します。

---

## おわりに：コードに「喜び」を取り戻す

CUPID特性の提唱者であるDan North氏は、**「Joyful Code（喜びのあるコード）」** という言葉を使っています。

SOLIDは偉大な先人の知恵であり、今も色あせない設計の羅針盤です。しかしAIが一瞬で数百行を生成する時代に、「この1行はSRPに反していないか」と一言一句チェックし続けることは、開発者の喜びを奪いかねません。

CUPIDはSOLIDを否定せず、**その意図を引き継ぎながら「AI時代の言葉」に翻訳した**ものです。

- AIへの指示に**CUPID制約を埋め込む**ことで、生成コードの品質を最初から高める
- **CUPIDチェックリスト**でAIに査読させ、レビューコストを下げる
- **CLAUDE.mdにIdiomatic・Domain語彙を常駐させ**、毎回の説明を省く

こうした実践を積み重ねることで、「AIに任せると設計が崩れる」という不安から解放され、開発者は本来集中すべき「**ドメインの問題を解くこと**」に向き合えるようになります。

皆さんのプロジェクトでも、AIと一緒に「このコード、もっとCUPID（愛の神）が必要じゃない？」と話し合ってみてはいかがでしょうか。

最後まで読んでいただき、ありがとうございました！

---

## 参考リンク

- **Dan North & Associates: CUPID Properties**
  - [CUPID—for joyful coding](https://dannorth.net/blog/cupid-for-joyful-coding/)
  - [Composable](https://dannorth.net/blog/cupid-for-joyful-coding/#composable) / [Unix Philosophy](https://dannorth.net/blog/cupid-for-joyful-coding/#unix-philosophy) / [Predictable](https://dannorth.net/blog/cupid-for-joyful-coding/#predictable) / [Idiomatic](https://dannorth.net/blog/cupid-for-joyful-coding/#idiomatic) / [Domain-based](https://dannorth.net/blog/cupid-for-joyful-coding/#domain-based)
- **Robert C. Martin: SOLID Principles**
  - [『アジャイルソフトウェア開発の奥義』](https://cdn.bookey.app/files/pdf/book/ja/%E3%82%A2%E3%82%B8%E3%83%A3%E3%82%A4%E3%83%AB%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E9%96%8B%E7%99%BA%E3%81%AE%E5%A5%A5%E7%BE%A9.pdf)（著：Robert C. Martin）
- **Python公式ドキュメント: typing.Protocol**
  - [typing — Support for type hints](https://docs.python.org/ja/3/library/typing.html#typing.Protocol)
- **Pydantic公式ドキュメント**
  - [Pydantic V2](https://docs.pydantic.dev/latest/)
