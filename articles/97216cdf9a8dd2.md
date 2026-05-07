---
title: "Amazon Bedrock Knowledge Bases 実装の落とし穴"
emoji: "🕳️"
type: "tech"
topics: ["aws", "bedrock", "rag", "knowledgebases", "bedrockkb"]
published: true
---

## はじめに

Amazon Bedrock Knowledge Bases (以下、KB) はフルマネージドな RAG 基盤として強力な選択肢ですが、本番運用に持っていこうとすると、公式ドキュメントだけでは拾いきれない落とし穴に遭遇します。

本記事は、**KB を実装した経験があり、運用課題に直面している中級者向け**に、構築フロー順にハマりどころを辞書的に引ける形でまとめた Tips 集です。各項目には可能な限り公式仕様の根拠リンクを添え、「なぜそういう挙動になるのか」の背景まで踏み込んで解説しています。

なお、本記事は AWS 公式情報および筆者の運用経験に基づくものです。AWS の最新仕様は随時変わる可能性があるため、実装時は必ず最新の公式ドキュメントもあわせてご確認ください。

### 取り上げるカテゴリ

1. データ取り込み編
2. メタデータ・フィルタリング編
3. 検索・リランク編
4. ネットワーク・セキュリティ編
5. コスト管理編

---

## 1. データ取り込み編

### 1.1 画像ファイル3.75MB上限の罠

KB にマルチモーダルデータを取り込む際、**JPEG/PNG 画像は 1 ファイルあたり 3.75MB** が上限です。テキストファイルの 50MB 上限の感覚で運用していると、画像だけが静かに失敗するケースに遭遇します。

> The maximum size of .JPEG and .PNG files is 3.75 MB.
> — [Prerequisites for your Amazon Bedrock knowledge base data](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-ds.html)

#### 何が起きるか

- データソース同期 (`IngestionJob`) の中で、対象画像のみが `FAILED` 扱いになる
- ジョブ全体は `COMPLETE` で終わるため、運用ログだけ見ると同期成功と誤認しやすい
- 失敗ファイルの詳細は `GetIngestionJob` API の `statistics.numberOfDocumentsFailed` および CloudWatch Logs で確認できる

#### なぜ 3.75MB なのか

KB のマルチモーダル取り込みは、内部的に **画像を Vision 対応の LLM (Claude などの VLM) に渡してテキスト/構造を抽出する** パーシングパイプラインを通っています。このとき画像は base64 エンコードされて InvokeModel API に乗るため、**Bedrock の InvokeModel リクエストペイロード上限と、base64 化による約 4/3 倍のサイズ膨張** が制約になります。

つまり、3.75MB という数字はストレージ側の制限ではなく、**取り込みパイプライン内部で呼ばれる Bedrock InvokeModel のペイロード制約から逆算された値** だと理解しておくと、なぜテキストの 50MB と桁違いに小さいのかが腑に落ちます。

#### 回避策

| 対策                   | 概要                                                              |
| ---------------------- | ----------------------------------------------------------------- |
| 事前バリデーション     | S3 アップロード前に Lambda などで画像サイズをチェックして弾く     |
| 画像の事前圧縮         | Pillow などで縮小・JPEG 品質調整してから取り込み                  |
| プレフィックス分離     | 画像用 / テキスト用 prefix を分け、画像側にだけサイズ監視を仕込む |
| そもそも別パイプライン | OCR / 図表抽出を Bedrock Data Automation 等の別経路に逃がす       |

#### 補足

テキストファイルは 1 ファイルあたり 50MB という制限があるため、相対的にずっと緩いです。同期ジョブのステータスだけを見て安心せず、必ず `numberOfDocumentsFailed` と CloudWatch Logs を確認するフローを運用に組み込むのが安全です(参考: [Knowledge Base SYNC Failed Issue - AWS re:Post](https://repost.aws/questions/QUlu4xPMCVQMKdoL0U3I-LwA/knowledge-base-sync-failed-issue))。

---

### 1.2 マルチモーダルKBのファイル数の罠

KB に Nova Multimodal Embeddings を選択したり、Bedrock Data Automation (BDA) を有効にすると、データソース内のファイル数が一定数を超えた時点で同期ジョブが完了しなくなります。

> Files per ingestion job: Maximum 15,000 files per job (Nova Multimodal Embeddings) or 1,000 files per job (BDA)
> — [Troubleshooting multimodal knowledge bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-multimodal-troubleshooting.html)

加えて、Claude 系の FM パーシング(高度なパーシング)を使った場合は、**「ジョブあたり総ファイルサイズ 100MB」** という別の上限が re:Post で報告されています ([Error while syncing knowledge base with Claude 3 Haiku v1 parsing strategy - AWS re:Post](https://repost.aws/questions/QUxNcHm_0hRJOB8aZMaUXVjA/error-while-syncing-knowledge-base-with-claude-3-haiku-v1-parsing-strategy))。こちらは公式ドキュメントには明記されていませんが、運用上は意識しておくべき値です。

#### 何が起きるか

- 同期ジョブが `IN_PROGRESS` のまま長時間進まない、あるいはタイムアウトで `FAILED`
- エラーメッセージとして `Maximum total file size limit for given parsing strategy : 104857600 reached` のような表示
- 「デフォルトのチャンキング+S3 同期」では問題なかったのに、FM パーサーや セマンティックチャンキング、BDA を有効化した瞬間に詰まる

#### なぜ上限が厳しくなるのか

通常のチャンキング(固定サイズ・階層的)は **テキスト処理だけで完結する軽い処理** ですが、FM パーサー/BDA/セマンティックチャンキングは **取り込みフェーズの中で都度 Bedrock の FM を呼び出す** 仕組みです。

- FM パーサー: ファイルごとに FM が「テキスト抽出 + 構造化」を行う
- セマンティックチャンキング: 段落間の意味的境界を FM の埋め込みベクトルで判定する
- BDA: 動画・音声・複雑な PDF を別パイプラインで解析する

つまり、ジョブ完了までに **大量の Bedrock 呼び出しが発生する** ため、内部的なスロットル・タイムアウト・コスト保護を兼ねて、ファイル数や合計サイズに上限が課されています。デフォルトの軽量チャンキングと同じ感覚で大量データを投入すると詰まるのは、この処理重さの差が背景にあります。

#### 回避策

実運用では **データソースを意図的に分割し、500〜1,000 ファイル単位で同期を分ける** 戦略が有効です。

```python
# 例: S3 inclusion prefix で論理的にデータソースを分ける
# data-source-A: s3://bucket/docs/2024/01/  (500 files)
# data-source-B: s3://bucket/docs/2024/02/  (500 files)
# ...
```

Terraform/Terragrunt 構成では、prefix ごとに `aws_bedrockagent_data_source` リソースを切る形になります。1 つの KB に複数のデータソースを紐づけられるので、データソース粒度での分割は仕様上自然です。

#### 補足:並列同期の上限

並列同期にも上限があります。AWS 公式ブログによると Bedrock の同期ジョブは以下の制約を持ちます ([Build and deploy an automatic sync solution for Amazon Bedrock Knowledge Bases](https://aws.amazon.com/blogs/machine-learning/build-and-deploy-an-automatic-sync-solution-for-amazon-bedrock-knowledge-bases/))。

| 制約                                           | 値                       |
| ---------------------------------------------- | ------------------------ |
| AWS アカウントあたりの同時 ingestion ジョブ数  | **5 並列**               |
| Knowledge base あたりの同時 ingestion ジョブ数 | 1 並列                   |
| Data source あたりの同時 ingestion ジョブ数    | 1 並列                   |
| `StartIngestionJob` API レート制限             | 0.1 req/秒(10 秒に 1 回) |

EventBridge 等で自動同期パイプラインを組む場合は、特に **アカウントあたり 5 並列** と **API レート 0.1 req/秒** の両方を意識した同時実行制御が必要です。

---

### 1.3 KBのデータソース上限の罠

KB の設計時に押さえておくべき構造的な上限として、**1 つの Knowledge Base に紐づけられるデータソースは 5 つまで**、加えて **1 データソースに含められるファイル数にも上限がある** という制約があります。1.2 で紹介した「データソース粒度で分割して同期負荷を下げる」戦略は、この 5 という上限の中で組む必要があります。

#### 公式仕様としての裏付け

[Amazon Bedrock の Service Quotas](https://docs.aws.amazon.com/general/latest/gr/bedrock.html) に「Data sources per knowledge base」の上限が明記されています。本記事執筆時点で **5 データソース / KB** がデフォルト値です(調整可能なソフトリミット)。

データソースあたりのファイル数上限はソース種別(S3 / Confluence / SharePoint / Salesforce / Web Crawler)ごとに個別に定められているため、最新値は同クォータページを参照してください。

#### 何が起きるか

- 1.2 の戦略で「ファイル数を分割するためにデータソースを増やしていく」と、6 つ目のデータソース追加で `ValidationException` が返る
- マルチテナント運用で「テナントごとにデータソースを切る」設計を素朴に組むと、5 テナントで上限に達する
- データソース内のファイル数上限を超えるサイズのデータセットを 1 データソースに集約しようとすると、同期ジョブが失敗する

#### なぜこの上限なのか

5 という数字の正確な意図は公式には説明されていませんが、**KB の同期パイプライン(ベクターインデックスへの書き込み・メタデータ整合性確保)の管理コストを抑えるため** の設計判断と推測できます。データソース数が増えるほど、KB 全体の同期ジョブ管理が複雑化するためです。

ソフトクォータなので Service Quotas コンソールから引き上げ申請は可能ですが、**「データソースを増やす」よりも「KB 自体を分割する」「データソース内をメタデータで論理分割する」ほうが運用しやすい** ケースが多いです。

#### 回避策

| 対策                                               | 概要                                                                | 推奨度                             |
| -------------------------------------------------- | ------------------------------------------------------------------- | ---------------------------------- |
| **業務ドメイン単位で KB 自体を分割**               | 「人事」「経理」「営業」など意味のあるドメイン単位で複数 KB を構築  | **推奨**                           |
| 1 データソース内で prefix/メタデータによる論理分割 | 物理的なデータソース数は増やさず、メタデータフィルタで分離          | テナント分離なら明示的フィルタ必須 |
| Service Quotas からのクォータ引き上げ申請          | データソース数の上限値を引き上げる(承認まで数営業日)                | 緊急対応・特殊要件向け             |

#### 補足

特に **マルチテナント運用ではテナント数が 5 を超えやすい** ため、初期設計の段階で「KB をテナント単位で分けるのか / 1 KB をメタデータで論理分割するのか」を決めておくことが重要です。後者を採る場合は、2.2 で触れた **明示的フィルタリング** によるテナント分離をセットで実装する必要があります。

---

### 1.4 自動作成IAMロールの罠

KB をマネジメントコンソールから作成すると、サービスロールと S3 アクセス用ポリシーが自動生成されます。一見便利ですが、**この自動作成ロールはコンソール経由の操作専用** であり、CLI/SDK/IaC を併用する運用ではトラブルの種になります。

#### 公式の見解

AWS re:Post の公式記事に明確な記述があります。

> The automatically generated execution role for knowledge base created by Bedrock service when using the AWS Console is **designed with specific IAM policies only for that role**. **When using CLI commands or SDK operations, it's recommended to create a custom role with appropriate permissions.**
> — [Bedrock Knowledge Base with S3 Vectors: Troubleshooting IAM Permission Issues - AWS re:Post](https://repost.aws/articles/ARBB7YtfrCRPGtBPnk09O_jA/bedrock-knowledge-base-with-s3-vectors-troubleshooting-iam-permission-issues)

つまり、**コンソールの自動作成ロールは「そのコンソール操作のためだけ」** に最適化されており、運用フェーズで CLI/SDK/IaC から使い回す前提では設計されていない、という公式スタンスです。

#### 何が起きるか

- 自動作成された S3 アクセスポリシーは **指定 prefix のみに権限を絞る** 設計のため、データソースを増やすたびに **prefix 単位で個別に書き加える形** になる
- IAM ロールのインラインポリシー上限は **10,240 文字**([IAM character limits - AWS re:Post](https://repost.aws/knowledge-center/iam-increase-policy-size))ですが、prefix ごとの ARN 列挙が積み上がるとこの上限に近づきやすい
- ロールやポリシーを手動編集すると、コンソール側の自動更新ロジックが意図しない挙動になる場合がある(筆者の運用経験上、構成変更時にポリシーが期待通りに追従しないケースを観測)
- CLI/SDK/IaC で KB を更新するときに、コンソールが想定していない経路からのアクセスで権限不足エラーが起きる

#### なぜこういう設計なのか

これは KB 固有の問題ではなく、**AWS のマネジメントコンソールが「最小権限の原則」に従って自動生成する** ことの副作用です。コンソールは指定された prefix のみにアクセスを絞ったポリシーを作る (= 必要十分な権限だがコンソール経由の構成変更に最適化) という設計になっています。

prefix 単位の絞り込みは安全側に倒した設計ですが、**「コンソール以外からも触られる可能性」を前提にしていない** ため、運用フェーズでの取り回しが悪くなります。

#### 回避策

実運用では **以下のいずれかのパターン** を選ぶのが現実的です。

**パターン A: 自動作成ロール/ポリシーを使うが触らない**

- ポリシーが必要十分であれば、何も手を入れずに使い続ける
- 構成変更はコンソール経由でのみ実施し、CLI/API/IaC で部分編集しない
- 短期検証なら最も手間が少ない

**パターン B: 自動作成ロール/ポリシーを削除し、バケット全体に権限を持つカスタムポリシーをアタッチ**

- KB 作成時に生成されたロールから、自動作成のインラインポリシーを削除
- 代わりに **バケット全体に対する `s3:GetObject` を許可するカスタマー管理ポリシー** をアタッチ
- prefix 追加のたびにポリシーを書き換える必要がなくなる

**パターン C: そもそもコンソールから作らず、IaC で初手から組む(推奨)**

- Terraform / CloudFormation で KB とロールを宣言的に構築
- 自動作成ポリシーが介在しないので、本問題自体が発生しない
- 公式の re:Post 記事も CLI/SDK 操作時は **カスタムロール作成を推奨** している
- 中長期運用ならこのパターンが最も安全

公式リファレンス: [Create a service role for Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-permissions.html) には、カスタムロール作成時に必要な権限ポリシーが網羅されています。

---

## 2. メタデータ・フィルタリング編

### 2.1 暗黙的フィルタとキーワードの罠

KB のメタデータフィルタリングは強力ですが、**暗黙的フィルタリング (Implicit Filtering)** を使う場合、メタデータに書いたキーワードが **本文 (チャンク) 内にも存在しないと、検索時にフィルタが期待通りに機能しない** という落とし穴があります。

> Implicit filtering allows you to automatically filter search results based on metadata attributes without requiring explicit filter expressions in each query.
> — [ImplicitFilterConfiguration - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_ImplicitFilterConfiguration.html)

#### 何が起きるか

- `<file>.metadata.json` でカテゴリ・タグを付与
- ユーザーが「カテゴリ X の文書を取って」とクエリすると、暗黙的フィルタリングが自動で `category=X` という条件を生成
- ところが、**本文中に X という単語が出てこない文書は、そもそもベクトル検索で上位に来ない** ため、フィルタ条件を満たす候補が 0 件になる
- 結果として「メタデータは付けたのに検索結果が出ない」現象が起きる

#### なぜそうなるのか (重要)

これは KB の **検索アーキテクチャ** を理解すると腑に落ちます。

KB のクエリは大きく **2 段階** で動いています。

1. **ベクトル検索ステージ**: クエリ文字列を埋め込みベクトル化し、チャンクのベクトルとコサイン類似度等で上位 N 件を取得
2. **メタデータフィルタリングステージ**: 取得した候補に対してメタデータ条件で絞り込み

ここで重要なのは、**メタデータはチャンク本文の埋め込みベクトルには含まれない** という点です。メタデータは「フィルタ条件を満たすかどうかの判定」にだけ使われ、「ベクトル類似度の計算」には寄与しません。

つまり、

- メタデータに `category: "金融"` と書いた
- 本文には「金融」という単語が一切出てこない
- ユーザーが「金融について教えて」と質問

このとき、ベクトル検索ステージで本文が上位に来ないので、その後にメタデータフィルタが「金融カテゴリだけ残す」と頑張っても候補がそもそもいない、という結果になります。

これは「**メタデータでセマンティックな意味を補強しているつもり**」という勘違いから起きる落とし穴で、ベクトル検索とメタデータフィルタの役割分担を誤解していると気づきにくい挙動です。

#### 回避策

| 対策                             | 概要                                            | 公式推奨度          |
| -------------------------------- | ----------------------------------------------- | ------------------- |
| 本文末尾にメタデータをコピー追記 | 本文に「カテゴリ: 金融」のような行を追記        | 非推奨 (運用負荷大) |
| **ハイブリッド検索の有効化**     | セマンティック検索 + キーワード検索の組み合わせ | **推奨**            |
| 明示的フィルタリングへの切り替え | クエリ時にフィルタ条件を API で明示             | 厳格な制御に有効    |

公式が推奨するのは **ハイブリッド検索** です。これは BM25 等のキーワード検索を併用するため、メタデータと完全一致するキーワードがクエリ側にあれば、本文中に出てこなくても拾える可能性が高まります。

ただし、ハイブリッド検索でも **本文と完全に乖離したメタデータ** は当然引っかかりませんので、「メタデータは本文の補助情報であって、独立したシグナルではない」という前提は変わりません。

#### 補足

「本文末尾にメタデータをコピー追記」は確実に動きますが、

- ドキュメント本文が冗長になる
- 元データの管理とインデックス用データの管理が二重化する
- メタデータ更新時に本文側も再生成が必要

といった運用負荷があるため、公式では推奨されていません。**短期的なワークアラウンド** として認識するのが適切です。

---

### 2.2 暗黙vs明示フィルタリングの罠

KB のメタデータフィルタリングには **暗黙的 (Implicit) と明示的 (Explicit) の 2 種類** があります。どちらを選ぶかで、エンドユーザーの体験と実装の堅牢性が変わります。

| 観点               | 暗黙的フィルタリング                                       | 明示的フィルタリング                |
| ------------------ | ---------------------------------------------------------- | ----------------------------------- |
| API 設定           | `ImplicitFilterConfiguration` で metadataAttributes を定義 | クエリ時に `RetrievalFilter` を指定 |
| エンドユーザー操作 | クエリ文字列のみ。フィルタは LLM が自動推論                | 明示的にフィルタ条件を指定          |
| LLM コール         | あり (フィルタ抽出用に追加 1 回発生)                       | なし                                |
| 制御の厳格性       | LLM の解釈次第でブレる                                     | 完全に決定論的                      |
| 推奨用途           | UX を重視するチャットボット等                              | 厳格な権限制御・テナント分離等      |

公式リファレンス: [RetrievalFilter](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrievalFilter.html), [ImplicitFilterConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_ImplicitFilterConfiguration.html)

#### なぜ 2 種類あるのか

両者は **トレードオフ関係** にあります。

- **暗黙的フィルタリング** はユーザーに優しい代わりに、LLM がメタデータの description を読んで「このクエリならこのフィルタを当てるべき」と推論するため、description の質に依存します。description が曖昧だと、フィルタが当たらなかったり、逆に意図しないフィルタが当たったりします。

- **明示的フィルタリング** はアプリ側が完全に制御できる代わりに、ユーザーから見えないコンテキスト (ログインユーザーのテナント ID など) をアプリ側で組み立てて渡す必要があります。

#### 厳格なフィルタ制御は明示的フィルタリングを推奨

特に **マルチテナントの権限制御** や **コンプライアンス要件のあるドキュメント分離** では、**明示的フィルタリングを必須**とすべきです。

理由は単純で、**暗黙的フィルタリングは「LLM がフィルタ条件を解釈する」というステップが入るため、プロンプトインジェクション等で意図しないテナントの文書を引き当てるリスクがある** からです。アプリ側で `tenant_id = $current_user.tenant` のようにハードコードして渡せば、LLM の解釈を介さないので安全です。

```python
# 明示的フィルタリングの例 (Python / boto3)
response = bedrock_agent_runtime.retrieve(
    knowledgeBaseId=kb_id,
    retrievalQuery={"text": user_query},
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "filter": {
                "andAll": [
                    {"equals": {"key": "tenant_id", "value": current_tenant_id}},
                    {"equals": {"key": "status", "value": "published"}},
                ]
            }
        }
    },
)
```

#### 暗黙的フィルタリングを使う場合の鉄則: description を磨く

暗黙的フィルタリングを採用する場合、**メタデータ属性の description フィールドの精度がそのまま検索品質を決めます。**

- 何を表す属性か (例: 「文書が公開された年。YYYY 形式」)
- 値の取りうる範囲 (例: 「`finance`, `legal`, `hr` のいずれか」)
- LLM がフィルタすべきクエリの典型パターン

を、**LLM が読んで判断できるレベルで明確に書く** ことが重要です。

---

## 3. 検索・リランク編

### 3.1 リランク入力サイズの罠

KB にリランクモデルを組み込むと、リランクステージで **入力サイズ超過のバリデーションエラー** が出るケースがあります。

#### 何が起きるか

- KB の `Retrieve` または `RetrieveAndGenerate` 呼び出し時に `ValidationException`
- エラーメッセージとして「入力サイズ超過」「token limit exceeded」等
- ベクトル検索で取得したチャンク数 × チャンクサイズが、リランクモデルの入力上限を超えると発生

#### なぜそうなるのか

リランクモデルにはそれぞれ **1 ドキュメントあたりのコンテキスト長** が設定されています。

| モデル             | 1 ドキュメントあたりのコンテキスト長 |
| ------------------ | ------------------------------------ |
| Cohere Rerank 3.5  | **4,096 トークン**                   |
| Cohere Rerank v4.0 | 32,768 トークン                      |

> Our rerank-v3.5 and rerank-v3.0 models, are trained with a context length of 4096 tokens (...)
> If your query is larger than 2048 token, it will be truncated to the first 2048 tokens (leaving the other 2048 for the document(s)).
> — [Best Practices for using Rerank | Cohere](https://docs.cohere.com/docs/reranking-best-practices)

これに加え、KB のリランク実装では **「ベクトル検索で取得した上位 N チャンクすべてをリランクに渡す」** という流れになります。N が大きく、かつ各チャンクが長いと、合計の入力サイズが内部的なペイロード上限を超えてエラーになります。

LLM 側 (RetrieveAndGenerate の生成ステージ) はモデルによって異なりますが、Claude 系であれば最大 200K トークンと余裕があるので、**ボトルネックはリランクモデル側のコンテキスト長になりやすい** という構図です。

#### 回避策

**リランクに渡すチャンク数を減らす** のが最も確実です。

```python
response = bedrock_agent_runtime.retrieve(
    knowledgeBaseId=kb_id,
    retrievalQuery={"text": user_query},
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "numberOfResults": 20,  # ベクトル検索で取得する数
            "rerankingConfiguration": {
                "type": "BEDROCK_RERANKING_MODEL",
                "bedrockRerankingConfiguration": {
                    "numberOfRerankedResults": 5,  # リランク後に残す数
                    "modelConfiguration": {
                        "modelArn": "arn:aws:bedrock:us-west-2::foundation-model/cohere.rerank-v3-5:0"
                    }
                }
            }
        }
    },
)
```

調整方針:

| パラメータ                       | 推奨アプローチ                                               |
| -------------------------------- | ------------------------------------------------------------ |
| `numberOfResults` (ベクトル検索) | リランクに渡す総量。チャンクサイズと相談して 20 前後から開始 |
| チャンクサイズ                   | 過大なら 300〜500 トークンに分割を見直す                     |
| `numberOfRerankedResults`        | LLM に渡す量。多くて 5〜10                                   |

#### 補足

リランクの最終結果が LLM (`RetrieveAndGenerate` の生成側) に渡される際は、別途その LLM のコンテキスト長制約が効きます。Claude 系なら最大 20,000 バイト程度のクエリ長を見ておけば実用上は十分です。リランク段階で詰まる場合は、まず **リランクに渡すチャンク数を減らすことから着手** するのが定石です。

---

## 4. ネットワーク・セキュリティ編

### 4.1 閉域KBのDashboardsの罠

KB のベクターストアに OpenSearch Serverless を選び、ネットワーク設定を **プライベート (Bedrock からの AWS service private access のみ許可)** にすると、**OpenSearch Dashboards に一切サインインできなくなります**。これは公式仕様です。

#### 公式仕様としての裏付け

OpenSearch Serverless のネットワークポリシーで、**AWS service private access (Bedrock など) を SourceServices に指定すると、Dashboards 側にはアクセス権限を付与できない** ことが公式ドキュメントに明記されています。

> **Private access to AWS services can only apply to the collection's OpenSearch endpoint, not to the OpenSearch Dashboards endpoint. AWS services cannot be granted access to OpenSearch Dashboards.**
> — [Overview of security in Amazon OpenSearch Serverless](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-security.html)

> Private access to AWS services such as Amazon Bedrock only applies to the collection's OpenSearch endpoint, not to the OpenSearch Dashboards endpoint. Even if the ResourceType is dashboard, AWS services cannot be granted access to OpenSearch Dashboards.
> — [Network access for Amazon OpenSearch Serverless](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-network.html)

#### 何が起きるか

- マネジメントコンソールで KB の「テスト」タブからクエリは実行できる (Bedrock 経由のアクセスは許可されているため)
- しかし、OpenSearch Serverless 側の Dashboards にサインインしようとすると 401/403 で弾かれる
- 結果として「同期成功した」と KB は言っているが、**実際にどのチャンクがインデックスされているか目視確認できない**

#### なぜこういう設計なのか

OpenSearch Serverless のネットワークポリシーには、

- **Collection endpoint への AWS service private access** (Bedrock など)
- **OpenSearch Dashboards endpoint への access** (VPC エンドポイント / パブリック)

の 2 つを別々に設定する必要があります。仕様上、**AWS service (Bedrock) は前者のみに権限を付与でき、Dashboards 側には付与できない** という制約が課されています。

これはセキュリティ設計上の意図的な分離で、Dashboards はブラウザベースの管理 UI なので「サービス間通信用の AWS service private access」という抽象には合わない、という判断と理解できます。

#### 回避策

**CLI / SDK 経由で OpenSearch Serverless に直接アクセス** するのが現実的です。

前提として、

- 適切な IAM ポリシー (`aoss:APIAccessAll`) がアタッチされていること
- VPC エンドポイントまたはアクセスポリシーで自身の IAM プリンシパルが許可されていること

```bash
# 例: aws-cli + awscurl で OpenSearch Serverless に直接クエリ
# (Bastion / EC2 上の VPC 内から実行)

ENDPOINT="https://xxxxxxxxx.us-east-1.aoss.amazonaws.com"
INDEX="bedrock-knowledge-base-default-index"

aws opensearchserverless list-collections --region us-east-1

# ドキュメント数の確認
awscurl --service aoss --region us-east-1 \
  -X GET "${ENDPOINT}/${INDEX}/_count"

# インデックス内のサンプル取得
awscurl --service aoss --region us-east-1 \
  -X POST "${ENDPOINT}/${INDEX}/_search" \
  -H "Content-Type: application/json" \
  -d '{"size": 5, "_source": ["AMAZON_BEDROCK_TEXT_CHUNK", "AMAZON_BEDROCK_METADATA"]}'
```

#### 補足

「どうしても Dashboards も使いたい」場合は、**Dashboards 側だけ別途ネットワークアクセスを付与** する設計にする手もあります。例えば、

- Collection endpoint: AWS service (Bedrock) からのみ private access
- Dashboards endpoint: 自社 VPC からの VPC エンドポイント経由でアクセス可能

という構成です。ただしこの場合は VPC エンドポイントの追加コストとセキュリティ設計の複雑化を伴います。

---

### 4.2 VPCエンドポイント5種の罠

Bedrock 関連の VPC エンドポイント (PrivateLink) は **5 種類存在し、それぞれ役割が違います**。閉域環境での実装時に「どれを置けばいいか」を取り違えると、見えないエラーで時間を溶かします。

> Create an interface endpoint for Amazon Bedrock using any of the following service names:
> — [Use interface VPC endpoints (AWS PrivateLink) ... Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html)

#### 5 種類のエンドポイントとその役割

| サービス名                                     | プレーン     | 用途                                          | 主な API                                               |
| ---------------------------------------------- | ------------ | --------------------------------------------- | ------------------------------------------------------ |
| `com.amazonaws.<region>.bedrock`               | コントロール | モデル/Provisioned Throughput 等の管理        | ListFoundationModels, CreateProvisionedModelThroughput |
| `com.amazonaws.<region>.bedrock-runtime`       | データ       | **モデル推論呼び出し**                        | InvokeModel, Converse, Rerank                          |
| `com.amazonaws.<region>.bedrock-mantle`        | データ       | **新世代の Responses / Chat Completions API** | CreateInference (Responses, Chat Completions)          |
| `com.amazonaws.<region>.bedrock-agent`         | コントロール | エージェント / KB の構成管理                  | CreateAgent, CreateKnowledgeBase, StartIngestionJob    |
| `com.amazonaws.<region>.bedrock-agent-runtime` | データ       | **エージェント / KB の呼び出し**              | InvokeAgent, Retrieve, RetrieveAndGenerate             |

#### なぜこれだけに分かれているのか

AWS のサービス設計の典型パターンとして、

- **コントロールプレーン**: リソースの作成・削除・設定変更 (低頻度・管理操作)
- **データプレーン**: 実際の処理リクエスト (高頻度・本番トラフィック)

を分離することで、**運用上の障害の分離** (片方が落ちてももう片方は動く) や **レート制限の独立管理** (管理操作で本番が詰まらない) を実現しています。

Bedrock の場合さらに、

- **基盤モデル系** (`bedrock`, `bedrock-runtime`, `bedrock-mantle`)
- **エージェント / KB 系** (`bedrock-agent`, `bedrock-agent-runtime`)

の 2 軸が掛け合わさって 5 種類になっている、という構造です。`bedrock-mantle` は OpenAI 互換の Responses / Chat Completions API および後述する Projects(コスト管理機能)で利用される新しいエンドポイントなので、新規構築時は最初から考慮に入れておくのが安全です。

#### 実装時のチェックリスト

| やりたいこと                                          | 必要なエンドポイント                        |
| ----------------------------------------------------- | ------------------------------------------- |
| KB を IaC で構築・更新する                            | `bedrock-agent`                             |
| KB に対してクエリ (Retrieve / RetrieveAndGenerate)    | `bedrock-agent-runtime` + `bedrock-runtime` |
| Claude などの LLM を直接呼ぶ (InvokeModel / Converse) | `bedrock-runtime`                           |
| Responses / Chat Completions API を呼ぶ               | `bedrock-mantle`                            |
| エージェントを呼び出す                                | `bedrock-agent-runtime` + `bedrock-runtime` |
| ガードレールを呼ぶ                                    | `bedrock-runtime` (ApplyGuardrail API)      |

特に **KB クエリは `bedrock-agent-runtime` だけでは足りず、内部で LLM を呼ぶため `bedrock-runtime` も必要** という点は、閉域環境で `RetrieveAndGenerate` が突然失敗するときの典型的な原因です。

#### 補足

KB が S3 を読みに行く経路も忘れずに。**S3 ゲートウェイエンドポイント** が VPC に必要です (Interface 型ではなく Gateway 型なので別物)。

---

### 4.3 ガードレール日本語対応の罠

「日本語ガードレール = STANDARD tier 必須 = クロスリージョン推論必須」と単純化されがちですが、**実際にはフィルタの種類ごとに対応言語と tier 概念が異なります**。日本語コンテンツを扱う際は、**どのフィルタを使うか** によってアーキテクチャ要件が変わります。

#### 公式仕様: フィルタ別の言語サポート

[Languages supported by Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-supported-languages.html) の公式表をもとに、日本語サポート状況を整理すると次のようになります。

| フィルタ種別                            | 日本語サポート                                      | tier 概念                       | クロスリージョン推論 |
| --------------------------------------- | --------------------------------------------------- | ------------------------------- | -------------------- |
| Content filters / Prompt attacks        | STANDARD tier のみ                                  | **あり** (Classic は英・仏・西) | **STANDARD は必須**  |
| Denied topics                           | STANDARD tier のみ                                  | **あり** (Classic は英・仏・西) | **STANDARD は必須**  |
| **Sensitive information filters (PII)** | **tier 区分なしで日本語 "Optimized and supported"** | **なし**                        | **不要**             |
| Word filters                            | サポート対象外(英・仏・西のみ)                      | なし                            | 不要                 |
| Contextual grounding checks             | サポート対象外(英・仏・西のみ)                      | なし                            | 不要                 |

> Standard tier ... Provides more robust performance compared to Classic tier and has more comprehensive language and code-related prompt support. ... **Guardrails with Standard tier also use cross-Region inference.**
> Classic tier ... Provides established guardrails functionality supporting English, French, and Spanish languages.
> — [Safeguard tiers for guardrails policies](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html)

#### 重要なポイント: PII フィルタは tier 概念の対象外

Sensitive information filters (PII) は **そもそも tier 区分の対象になっていません**。日本語を含む 17 言語が "Optimized and supported" として明記されており、**シングルリージョン (= 日本リージョン閉域) でも問題なく動作** します。

これに対して **Content filters / Denied topics は tier 区分の対象** であり、

- Classic tier: 英・仏・西の 3 言語のみ
- STANDARD tier: 日本語含む多言語(クロスリージョン推論必須)

という設計になっています。

つまり、**「日本語ガードレールを使いたい」と一言で言っても、何を保護したいかによって要件が大きく変わる** わけです。

#### なぜ tier 概念に含まれるフィルタと含まれないフィルタがあるのか

これは推測になりますが、

- **PII フィルタ**: パターンマッチングと文脈判定の組み合わせで、エンティティ抽出は比較的軽いタスク。多言語に拡張しやすい
- **Content filters / Denied topics**: 文意の判断・ハルシネーション検出など、より大きなモデルが必要。多言語化はモデル容量の増加につながり、その結果としてキャパシティ確保が難しくクロスリージョン推論で需要分散

という設計判断があると考えられます。Sensitive information filters はそもそもの計算負荷が低いため、**全リージョン・全 tier で日本語含む多言語が標準サポート** されている、と理解するのが整合的です。

#### 結論: ユースケース別の選択肢

| ユースケース                                    | 推奨構成                                                 | 注意点                                              |
| ----------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------- |
| **日本リージョン閉域 + 日本語 PII マスキング**  | PII フィルタのみのガードレール (シングルリージョン)      | tier 設定不要、`bedrock-runtime` の VPCE のみで完結 |
| 日本リージョン閉域 + 日本語コンテンツフィルタ   | **両立不可**(STANDARD tier はクロスリージョン必須のため) | コンプライアンス要件と機能要件のいずれかを譲る      |
| クロスリージョン許容 + 日本語コンテンツフィルタ | STANDARD tier + APAC ガードレールプロファイル            | リージョン外データ移動の許諾を確認                  |

#### IAM 権限の落とし穴(クロスリージョン推論時)

クロスリージョン推論を使う場合、**プライマリリージョンだけでなく、ルーティング先の全リージョンに対する IAM 権限** が必要です。

[APAC ガードレールプロファイル](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-cross-region-support.html)では、`ap-northeast-1` (東京) をソースリージョンとして指定した場合、以下の宛先リージョンへルーティングされる可能性があります。

| ソースリージョン        | 宛先リージョン                                                                                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ap-northeast-1` (東京) | `ap-south-1` (ムンバイ), `ap-northeast-2` (ソウル), `ap-northeast-3` (大阪), `ap-southeast-1` (シンガポール), `ap-southeast-2` (シドニー), `ap-northeast-1` (東京) |

したがって IAM 権限はこれら全宛先リージョンを網羅する必要があります。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["bedrock:ApplyGuardrail", "bedrock:InvokeModel"],
      "Resource": [
        "arn:aws:bedrock:ap-northeast-1:*:guardrail/*",
        "arn:aws:bedrock:ap-northeast-1:*:foundation-model/*",
        "arn:aws:bedrock:ap-northeast-2:*:foundation-model/*",
        "arn:aws:bedrock:ap-northeast-3:*:foundation-model/*",
        "arn:aws:bedrock:ap-southeast-1:*:foundation-model/*",
        "arn:aws:bedrock:ap-southeast-2:*:foundation-model/*",
        "arn:aws:bedrock:ap-south-1:*:foundation-model/*"
      ]
    }
  ]
}
```

詳細は [Permissions for using cross-Region inference with Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrail-profiles-permissions.html) を参照。

---

## 5. コスト管理編

### 5.1 LLMコスト按分の罠

「テナント別 / アプリケーション別 / 機能別に Bedrock のコストを按分したい」というニーズには、**アプリケーション推論プロファイル (Application Inference Profile, AIP)** が公式の答えです。

> Application inference profiles (AIPs) let you attribute Amazon Bedrock costs by application, team, or workload (...) Each AIP is model-specific and carries cost allocation tags that flow to AWS Cost Explorer and AWS Cost and Usage Reports (CUR 2.0).
> — [Application inference profiles - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-application-inference-profiles.html)

#### なぜ AIP が必要なのか

オンデマンドの基盤モデルそのものには **タグを付けられません**。`anthropic.claude-sonnet-4-5` のような ARN は AWS 側のグローバルリソースなので、ユーザーがタグ付けする対象として存在しないためです。

> Note: You can't assign tags to on-demand models.
> — [Add cost allocation tags to Amazon Bedrock on-demand models - AWS re:Post](https://repost.aws/knowledge-center/bedrock-add-cost-allocation-tags)

そこで AIP は **「特定のモデル ARN を参照する、ユーザーアカウント内のリソース」** として作成可能で、こちらにはタグを付けられます。InvokeModel/Converse の `modelId` に AIP の ARN を指定すれば、その呼び出し分のコストが AIP のタグで分類されて Cost Explorer に流れる、という仕組みです。

#### KB の RetrieveAndGenerate でも使える

AIP は単体の LLM 呼び出しだけでなく、**KB の `RetrieveAndGenerate` API でも利用可能** です。

> Knowledge base vector embedding and response generation – Use an inference profile when generating a response after querying a knowledge base or when parsing non-textual information in a data source.
> — [Set up a model invocation resource using inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)

これにより、「テナント A の KB クエリ」と「テナント B の KB クエリ」を別の AIP で分けておけば、Cost Explorer 上でコストを分離できます。

#### Terraform 例

```hcl
resource "aws_bedrock_inference_profile" "tenant_a" {
  name        = "kb-tenant-a-claude-sonnet-4-5"
  description = "Tenant A 向け KB 用推論プロファイル"

  model_source {
    copy_from = "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-5-20250929-v1:0"
  }

  tags = {
    Tenant      = "tenant-a"
    CostCenter  = "kb-rag"
    Environment = "production"
  }
}
```

```python
# RetrieveAndGenerate 側で AIP の ARN を modelArn に渡す
response = bedrock_agent_runtime.retrieve_and_generate(
    input={"text": user_query},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": kb_id,
            "modelArn": tenant_a_aip_arn,  # AIP の ARN
        },
    },
)
```

#### 注意点

- AIP は **モデルごとに 1 つ必要**。10 モデル × 10 テナント = 100 AIP 作成、というように爆発しやすい
- タグはアクティベート (Billing コンソールで「コスト配分タグ」として有効化) しないと Cost Explorer に出ない
- アクティベート後 **24 時間程度の遅延** あり、かつ過去分には遡及しない (アクティベート以降の使用分のみタグ付き)

#### 補足

近年は AIP に加えて **Projects** (`bedrock-mantle` エンドポイントの Responses / Chat Completions API 向け) という新しい仕組みも登場しています。エンドポイントによって使い分けが必要なので、最新ドキュメント [Application inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-application-inference-profiles.html) で API 側の対応状況を確認するのがよいでしょう。

---

### 5.2 AOSSタグのコスト按分の罠

KB のベクターストアとして OpenSearch Serverless (AOSS) を使っている場合、**コレクションにコスト配分タグを付けても、Cost Explorer 上でタグ別の按分が期待通りにできない** という落とし穴があります。

> The issue you're experiencing with OpenSearch Service Serverless tags not appearing in Cost Explorer is a known concern.
> — [OpenSearch Service Serverless tags does not appear in Cost Explorer - AWS re:Post](https://repost.aws/questions/QU9FOD8EQBSD663uHFIpaK1A/opensearch-service-serverless-tags-does-not-appear-in-cost-explorer)

#### 何が起きるか

- AOSS のコレクションに `Tenant: tenant-a` のようなタグを付与
- Billing コンソールでコスト配分タグとしてアクティベート
- しかし Cost Explorer で「タグ別 OpenSearch コスト」を見ると、コレクションのタグが反映されない / 不完全

#### なぜそうなるのか

これは AOSS の **課金モデル** に起因します。

AOSS の料金は **OCU (OpenSearch Compute Unit)** という単位で課金されますが、OCU は **コレクション単位ではなくアカウント全体で割り当て・共有される** 仕組みです。最低でも 4 OCU (Indexing 2 + Search 2) がアカウント単位で常時確保され、コレクションを増やしてもこの OCU を分け合う形になります。

つまり、**請求発生主体が「個別のコレクション」ではなく「アカウントのプール OCU」** なので、コレクションに付けたタグが課金行 (line item) にマッピングされず、Cost Explorer 上では「タグなし」として現れるという挙動になります。

設計上の意図としては、AOSS は「サーバーレスでスケールするインフラ」を強調しており、**「マルチテナント前提のリソースに、個別のテナント単位の課金を割り当てるのは原理的に難しい」** という事情があります (cf. Lambda の Provisioned Concurrency など、近い思想のサービスも同様の制約を持つことが多い)。

#### 回避策

| アプローチ                                              | 概要                                                               | トレードオフ                                     |
| ------------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------ |
| **コレクションをテナントごとに分離**                    | 物理的に AOSS コレクションをテナント別に作る                       | 最低 4 OCU × テナント数のコストが発生 (高コスト) |
| **Aurora pgvector / OpenSearch Managed Cluster へ移行** | リソース単位の課金が明確なベクターストアを採用                     | 移行コスト + 性能特性の違い                      |
| **タグではなく、運用側で按分計算**                      | CUR + アプリログから利用量を集計し、自前で割り当て                 | 実装負荷が大きいが正確                           |
| **AIP (前項) と組み合わせて LLM コストだけ按分**        | ベクターストアコストは共通費として扱い、LLM コストのみテナント按分 | 折衷案。実装が比較的簡単                         |

#### 補足: OpenSearch Serverless Collection Groups

2026 年 2 月 10 日に GA された **OpenSearch Serverless Collection Groups** ([Amazon OpenSearch Serverless now supports Collection Groups - AWS What's New](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-opensearch-serverless-supports-collection-groups/)) という機能を使うと、複数コレクション間で OCU を共有できるようになり、マルチテナント構成のコスト最適化が改善されます。

主要な特徴は以下のとおりです。

- **異なる KMS キーを持つコレクション間で OCU を共有** できる(これが最大の価値提案)
- Collection Group 単位で **min / max OCU の双方** を設定可能(コールドスタート遅延を排除できる)
- 既存コレクションは Collection Group に関連付け不可、**新規作成のコレクションのみ** 関連付け可能
- 同一 Collection Group 内のコレクションは **同一タイプ (search / time series / vector search) のみ** 混在可能

これとタグを組み合わせることで、ある程度マルチテナントのコスト分離が改善される可能性がありますが、本記事執筆時点では Cost Explorer での完全なタグ別表示までは未確認です。

---

## おわりに

Bedrock Knowledge Bases は、フルマネージドであるがゆえに「内部の挙動」が見えにくく、想定外の制約に当たるとデバッグが難航しがちです。本記事の Tips は筆者が実際に運用フェーズで踏んだ落とし穴をベースにしていますが、それぞれの背景には **「サービスの内部アーキテクチャ」「課金モデル」「セキュリティとユーザビリティのトレードオフ」** といった設計判断が透けて見えます。

挙動を覚えるだけでなく **「なぜそうなっているのか」を理解しておくと、新しい類似ケースに遭遇したときに自力で原因を推定しやすくなります**。本記事がその一助になれば幸いです。

### 主要な公式ドキュメント (まとめ)

- [Amazon Bedrock Knowledge Bases - Prerequisites](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-ds.html)
- [Multimodal KB Troubleshooting](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-multimodal-troubleshooting.html)
- [Build and deploy an automatic sync solution for Amazon Bedrock Knowledge Bases (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/build-and-deploy-an-automatic-sync-solution-for-amazon-bedrock-knowledge-bases/)
- [Create a service role for Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-permissions.html)
- [IAM character limits - AWS re:Post](https://repost.aws/knowledge-center/iam-increase-policy-size)
- [ImplicitFilterConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_ImplicitFilterConfiguration.html)
- [RetrievalFilter](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrievalFilter.html)
- [Cohere Rerank Best Practices](https://docs.cohere.com/docs/reranking-best-practices)
- [Bedrock VPC Interface Endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html)
- [Network access for Amazon OpenSearch Serverless](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-network.html)
- [Overview of security in Amazon OpenSearch Serverless](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-security.html)
- [Languages supported by Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-supported-languages.html)
- [Safeguard tiers for guardrails policies](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html)
- [Supported Regions for cross-Region guardrail inference](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-cross-region-support.html)
- [Application Inference Profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-application-inference-profiles.html)
- [Amazon OpenSearch Serverless now supports Collection Groups](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-opensearch-serverless-supports-collection-groups/)

最新仕様は変動しますので、記事の情報を参考にされる際は必ずご自身で公式ドキュメントの最新版もご確認ください。本記事への指摘・追加情報は歓迎です。
