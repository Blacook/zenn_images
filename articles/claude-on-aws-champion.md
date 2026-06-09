---
title: "Claude on AWS Champion に認定された件"
emoji: "🥇"
type: "idea"
topics: ["claude", "anthropic", "aws", "claudecode", "bedrock"]
published: true
published_at: "2026-06-26 12:00"
---
## はじめに

2026年6月、**Claude on AWS Champion Program** に認定されました。

本記事では、このプログラムが何であるかの説明と、5つの認定要件をどのように達成したかの記録、そしてチャンピオンとして今後取り組んでいきたいことをまとめます。

---

## Claude on AWS Champion Programとは

Claude on AWS Champion Program は、**AnthropicとAWSの協業によるパートナー向けの認定プログラム**です。

Claude を活用したシステムの設計・実装・普及において実績を持つエンジニアを認定し、Anthropic 本社とのより密接な情報交換や最新情報へのアクセスを可能にします。

認定には以下の5つの要件を満たす必要があります。

| #   | 要件             | 内容                                                   | MoSCoW |
| --- | ---------------- | ------------------------------------------------------ | :----- |
| 1   | 資格取得         | Claude Certified Architect – Foundations（CCAF）の取得 | Must   |
| 2   | トレーニング受講 | 指定の Anthropic 公式トレーニング2コース               | Must   |
| 3   | 顧客実績         | Claudeを用いたシステムの提案・実装経験 1件以上         | Should |
| 4   | イネーブルメント | Claude 関連のセッションやワークショップの実施 2回以上  | Should |
| 5   | アウトプット     | ブログ・勉強会資料・登壇など 3本以上                   | Should |

---

## 認定要件の達成内容

### 1. 資格取得

**Claude Certified Architect – Foundations（CCAF）** を 2026年6月5日に取得しました。

CCAF は Anthropic が提供する Claude 公式認定試験で、Claude API・Claude Code を使ったシステム設計の知識を問います。受験経緯や勉強法については以下の記事にまとめています。

https://zenn.dev/blacook/articles/ccaf-exam-report

また、**AWS Certified Generative AI Developer – Professional** も 2026年1月31日に取得済みです。

:::message
CCAF が「Claudeを使いこなす設計力」を問うのに対し、AWS Certified Generative AI Developer – Professional は「AWS上でGenerative AIを実装する技術力」を問います。2つの資格が Claude × AWS の両軸をカバーしており、チャンピオン要件と整合していると感じています。
:::

---

### 2. トレーニング受講

Anthropic が提供する以下の公式トレーニングを受講しました。

- **Claude in Amazon Bedrock**（Anthropic Skilljar）
- **Claude Code in Action**（Anthropic Skilljar）

加えて、2026年5月にサンフランシスコで開催された **Anthropic Partner Bootcamp** に参加しています。エージェント工学のハンズオンを2日間で集中的に学ぶもので、そこで得た知見は以下の記事にまとめました。

https://zenn.dev/blacook/articles/agent-engineering-principles

---

### 3. 顧客に対するClaude活用実績

現在、以下の2件で顧客向けの提案・実装に取り組んでいます。

**① Strands Agents + Claude on Bedrock を用いたマルチエージェントアプリ構築**

Amazon Bedrock 上の Claude を呼び出すマルチエージェントアプリを、AWS が提供する [Strands Agents](https://strandsagents.com/) フレームワークを使って構築中です。複数のエージェントを疎結合に組み合わせ、タスクを分業処理する構成を採用しています。

**② AWS VPC 閉域内で Claude Code を開発業務の支援提案**

AWS VPC内のエアギャップ環境でClaude Code を活用して開発資産の構造解析とリファクタリング計画の立案を支援するアプローチを提案しています。コードの依存関係の可視化・技術的負債の特定・移行戦略の整理などを Claude と協働で進めます。

https://zenn.dev/blacook/articles/claude-code-bedrock-airgap-vpc

---

### 4. イネーブルメントセッション

**① Anthropic Partner Bootcamp 報告会**（全社任意・200人規模）

Bootcamp 参加直後に全社向けの報告会を実施しました。テーマは「how to harness agents」、すなわち **エージェントを作ることよりも、まず評価できるようになること** の重要性です。LLM システムの失敗の多くはモデルの問題ではなくプロンプト・ツール設計・コンテキスト管理の問題であり、症状を再現できる最小入力を作って層を一つずつ排除する診断アプローチが実務で効くというメッセージを伝えました。

**② プロジェクト内 Claude 勉強会**（任意・20人程度）

プロジェクト内のエンジニア向けに、初心者向け Claude Code の使い方をハンズオン形式で実施しました。Claude Code の基本操作・効果的な指示の出し方・セキュリティ上の注意点などを実演しながら解説しました。

**③ クライアント向け AI エージェント勉強会**（任意・10人程度）

クライアント向けに、AI エージェントの基礎概念や　Strands SDK を利用したマルチエージェント構築の設計原則などを具体的なソースコードを提示しながら解説しました。

---

### 5. アウトプット

認定時点でClaude 関連の記事を以下 4 本について公開しています。

| 記事                                                                                                                                         | 概要                                                                     |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| [Claude Codeの全機能を「飲食店の厨房」で完全理解する](https://zenn.dev/blacook/articles/claude-code-kitchen-analogy)                         | Claude Code の各機能を飲食店の厨房になぞらえて体系的に解説               |
| [VPC閉域（エアギャップ）環境のcode-serverでClaude CodeをBedrock経由で使う](https://zenn.dev/blacook/articles/claude-code-bedrock-airgap-vpc) | インターネット非接続の閉域環境で Claude Code を Bedrock 経由で動かす手順 |
| [Anthropic Partner Bootcampに参加した件](https://zenn.dev/blacook/articles/agent-engineering-principles)                                     | Bootcamp で得たエージェント設計の6原則の整理                             |
| [Claude Certified Architect – Foundations（CCAF）に合格した件](https://zenn.dev/blacook/articles/ccaf-exam-report)                           | CCAF の勉強法・試験傾向・判断基準の解説                                  |

記事の傾向として、**「エンタープライズの閉域環境で Claude を使う」** というテーマが多くなっています。日本の大企業では AWS の VPC 閉域内でシステムを運用するケースが多く、そこで Claude を安全に活用する方法は実務上のニーズが高い領域です。

---

## チャンピオンとして今後していきたいこと

### 1. AWS 閉域内での Claude Code 活用推進

日本のエンタープライズ環境では、セキュリティポリシーの制約から外部インターネットへの通信が制限されているケースが多くあります。そうした閉域環境でも Claude Code を安全かつコストリーズナブルに使えるアーキテクチャを実証し、パターン化して広めていきたいと考えています。

### 2. Anthropic 本社との交流・日本への情報発信

チャンピオンとして Anthropic 本社との定期的な交流機会を活かし、Claude の最新動向・ロードマップ・ベストプラクティスをいち早く日本に発信していきます。英語の技術情報をそのまま伝えるのではなく、日本のエンタープライズ文脈に合わせた解釈と実践例を添えて届けることを意識します。

### 3. 新しい領域での Claude Code 展開

現在取り組んでいる開発資産の解析・リファクタリングのほか、ドキュメント整備・提案書作成・プロジェクト管理など、Claude Code の適用領域を継続的に広げていきます。「Claude Code で何ができるか」を実体験ベースで語れるエンジニアとして、プロジェクトへの導入を支援していきます。

---

## おわりに

Claude on AWS Champion Program の認定は、個人としての活動の区切りではなく、スタート地点です。Claude を使いこなすエンジニアが増えることで、日本のソフトウェア開発、ひいては経済活動全般の生産性が上がると本気で思っています。これからも手を動かしながら発信を続けていきます。
