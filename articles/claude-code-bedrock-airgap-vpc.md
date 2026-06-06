---
title: VPC閉域（エアギャップ）環境のcode-serverでClaude CodeをBedrock経由で使う
emoji: "🔒"
type: "tech"
topics: ["claudecode", "bedrock", "extension", "vscode", "security"]
published: true
---

## はじめに

エンタープライズ環境では、セキュリティ要件によりVPCからインターネットへのアウトバウンド通信を完全に遮断した「エアギャップ環境」で開発を行うケースがあります。

本記事では、以下の環境でClaude Codeを利用する手順を解説します。

| 項目         | 環境                                            |
| ------------ | ----------------------------------------------- |
| IDE          | code-server（ブラウザベースVS Code）            |
| アクセス方式 | EC2リモートデスクトップ（RDP）                  |
| ネットワーク | VPC閉域網（NAT Gateway / Internet Gatewayなし） |
| LLMアクセス  | Amazon Bedrock（VPC Endpoint経由）              |
| IAM認証      | EC2 Instance Profile                            |

Claude Codeは公式にAmazon Bedrock経由の利用をサポートしており[^1]、Bedrock使用時はAnthropicへの認証通信（`api.anthropic.com`等）が不要になります[^2]。そのため、VPC Endpoint（PrivateLink）さえ構築すれば、エアギャップ環境でもLLM推論が可能です。

ただし、Claude Codeは推論以外にもテレメトリやアップデートチェック等の外部通信を行うため、それらを適切に抑止する設定が必要です。本記事ではその具体的な手順を、設定値の公式ドキュメント根拠とともに示します。

[^1]: [Claude Code on Amazon Bedrock - Claude Code Docs](https://code.claude.com/docs/en/amazon-bedrock)

[^2]: 同上 — Bedrock使用時は`/login`と`/logout`コマンドは無効化され、認証はAWSクレデンシャルで処理される

## アーキテクチャ概要

エアギャップ環境でClaude Codeを動かすには、3つのレイヤーを整理する必要があります。

### レイヤー1: LLM推論経路

Bedrock VPC Endpointを経由してモデルを呼び出します。Instance ProfileによるIAM認証で、APIキーの管理は不要です。

```
code-server (EC2)
  └── Claude Code拡張機能
        └── Bedrock SDK (SigV4署名)
              └── VPC Endpoint (PrivateLink)
                    └── bedrock-runtime.<region>.amazonaws.com
```

### レイヤー2: 非推論の外部通信

Claude Codeはデフォルトで以下のURLにアクセスします[^3]。

- `api.anthropic.com` — Claude APIエンドポイント（Bedrock使用時は不要）
- `claude.ai` / `platform.claude.com` — 認証（Bedrock使用時は不要）
- `storage.googleapis.com` / `downloads.claude.ai` — インストーラ・自動アップデート

エアギャップ環境ではこれらの通信を環境変数で明示的に無効化します。

[^3]: [Enterprise network configuration - Claude Code Docs](https://code.claude.com/docs/en/network-config)

### レイヤー3: インストール・配布

完全遮断環境では、VSIXファイルおよびCLIバイナリを事前に持ち込む必要があります。

## Step 0: 前提条件の確認

EC2にSSHまたはRDPで接続した状態から始めます。

```bash
# Instance Profileがアタッチされているか確認
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
# → ロール名が返ればOK

# リージョン確認
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
echo ${REGION}

# AWS CLIでcaller-identityが取れるか（STS経路の確認）
aws sts get-caller-identity
```

:::message alert
`aws sts get-caller-identity`が失敗する場合、STS用VPC Endpointが未構築の可能性があります。Instance Profileの一時クレデンシャル取得はIMDS（169.254.169.254）経由ですが、SDK操作の一部でSTSエンドポイントへの通信が発生します。
:::

## Step 1: VPC Endpoint経路の疎通確認

Claude Codeが必要とするIAMアクションは`bedrock:InvokeModel`、`bedrock:InvokeModelWithResponseStream`、`bedrock:ListInferenceProfiles`です[^4]。対応するVPC Endpointが正しく構成されていることを確認します。

[^4]: [Claude Code on Amazon Bedrock - IAM configuration](https://code.claude.com/docs/en/amazon-bedrock#iam-configuration)

### 必要なVPC Endpoint一覧

| サービス        | エンドポイント名                         | 用途                      |
| --------------- | ---------------------------------------- | ------------------------- |
| Bedrock Runtime | `com.amazonaws.<region>.bedrock-runtime` | モデル推論（InvokeModel） |
| Bedrock         | `com.amazonaws.<region>.bedrock`         | プロファイル一覧取得等    |
| STS             | `com.amazonaws.<region>.sts`             | クレデンシャル関連操作    |

### 疎通確認コマンド

```bash
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

# bedrock-runtime エンドポイント疎通（InvokeModel用）
nslookup bedrock-runtime.${REGION}.amazonaws.com
curl -s -o /dev/null -w "%{http_code}" \
  https://bedrock-runtime.${REGION}.amazonaws.com/

# bedrock エンドポイント疎通（ListInferenceProfiles用）
nslookup bedrock.${REGION}.amazonaws.com
curl -s -o /dev/null -w "%{http_code}" \
  https://bedrock.${REGION}.amazonaws.com/

# STS エンドポイント疎通
nslookup sts.${REGION}.amazonaws.com
```

## Step 2: IAM権限とBedrock疎通の確認

### 推論プロファイルの確認

```bash
# Claudeモデルが利用可能か確認
aws bedrock list-inference-profiles --region ${REGION} \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId, 'anthropic')].[inferenceProfileId, modelArn]" \
  --output table
```

### 推論テスト

```bash
aws bedrock-runtime invoke-model \
  --region ${REGION} \
  --model-id "us.anthropic.claude-sonnet-4-6" \
  --content-type "application/json" \
  --accept "application/json" \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":50,"messages":[{"role":"user","content":"Hello"}]}' \
  /tmp/bedrock-test-output.json

cat /tmp/bedrock-test-output.json
```

レスポンスが返ればBedrock推論経路は正常です。403エラーの場合はIAMポリシーまたはAnthropicのFTU（First Time Use）フォーム未提出の可能性があります。

### 必要なIAMポリシー

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowModelAndInferenceProfileAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock:ListInferenceProfiles"
      ],
      "Resource": [
        "arn:aws:bedrock:*:*:inference-profile/*",
        "arn:aws:bedrock:*:*:application-inference-profile/*",
        "arn:aws:bedrock:*:*:foundation-model/*"
      ]
    },
    {
      "Sid": "AllowMarketplaceSubscription",
      "Effect": "Allow",
      "Action": [
        "aws-marketplace:ViewSubscriptions",
        "aws-marketplace:Subscribe"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:CalledViaLast": "bedrock.amazonaws.com"
        }
      }
    }
  ]
}
```

出典: [Claude Code on Amazon Bedrock - IAM configuration](https://code.claude.com/docs/en/amazon-bedrock#iam-configuration)

## Step 3: 拡張機能のオフラインインストール

完全遮断環境では、VS Code Marketplace（`marketplace.visualstudio.com`）に到達できないため、VSIXファイルを事前に持ち込む必要があります。

### インターネットのある端末で事前準備

```bash
# 方法1: VS Code CLIでダウンロード
code --install-extension anthropic.claude-code --force
# ~/.vscode/extensions/ 配下にインストールされたファイルを取得

# 方法2: Open VSX Registryから取得（code-serverのデフォルトレジストリ）
# https://open-vsx.org/ で "Claude Code" を検索しVSIXをダウンロード

# 方法3: VS Code Marketplace API経由で直接ダウンロード
# https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code
# → "Download Extension" リンクからVSIXを取得
```

### EC2上でインストール

```bash
# S3経由やSCPでVSIXファイルをEC2に転送した後
code-server --install-extension /path/to/anthropic.claude-code-*.vsix
```

:::message alert
**最大のリスクポイント:** Claude Code拡張機能は初回起動時にCLIバイナリを`storage.googleapis.com`や`downloads.claude.ai`からダウンロードしようとする場合があります[^3]。エアギャップ環境ではこの通信が失敗し、拡張機能が起動しない可能性があります。

**対策:** インターネットのある検証環境で一度拡張機能を起動し、ダウンロードされるCLIバイナリ一式を特定してAMIにベイクしておくのが安全策です。
:::

```bash
# CLIバイナリの配置先を確認
find ~/.local/share/code-server/extensions/ -name "claude" -type f 2>/dev/null
ls -la ~/.claude/local/ 2>/dev/null
```

## Step 4: Bedrock向け設定の投入

設定ファイルは2箇所あります。それぞれの役割と配置場所を正確に把握することが重要です。

### 設定ファイルの対応表

| 設定ファイル            | パス                                            | 役割                                      |
| ----------------------- | ----------------------------------------------- | ----------------------------------------- |
| Claude Code設定         | `~/.claude/settings.json`                       | Claude Code本体の動作設定[^5]             |
| code-serverユーザー設定 | `~/.local/share/code-server/User/settings.json` | VS Code拡張機能の設定[^6]                 |
| code-serverサーバー設定 | `~/.config/code-server/config.yaml`             | code-server自体の起動設定（本件と無関係） |

[^5]: [Claude Code Settings](https://code.claude.com/docs/en/settings) — `settings.json`はClaude Codeの公式設定ファイルで、ユーザー設定は`~/.claude/settings.json`で定義される

[^6]: [code-server FAQ](https://coder.com/docs/code-server/FAQ) — VS Code設定やキーバインドはデフォルトで`~/.local/share/code-server`に保存される

:::message
code-serverの`--user-data-dir`オプションを変更している場合は、`<user-data-dir>/User/settings.json`がVS Code拡張機能側のパスになります。Coder経由で起動している場合はテンプレート側で上書きされていることもあるため、`ps aux | grep code-server`で実際の起動オプションを確認してください。

UIからも確認可能です: `Ctrl+Shift+P` → `Preferences: Open User Settings (JSON)` で開くファイルが該当します。
:::

### Claude Code設定（~/.claude/settings.json）

```bash
mkdir -p ~/.claude
cat > ~/.claude/settings.json << 'EOF'
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_REGION": "ap-northeast-1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "us.anthropic.claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "us.anthropic.claude-haiku-4-5-20251001-v1:0",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
EOF
```

### 各設定値の根拠

| 環境変数                                   | 用途                                          | 出典                                                                                                        |
| ------------------------------------------ | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `CLAUDE_CODE_USE_BEDROCK`                  | Bedrock統合の有効化                           | [Amazon Bedrock - Claude Code Docs](https://code.claude.com/docs/en/amazon-bedrock#3-configure-claude-code) |
| `AWS_REGION`                               | Bedrockエンドポイントのリージョン指定（必須） | 同上                                                                                                        |
| `ANTHROPIC_DEFAULT_SONNET_MODEL`           | モデルバージョンの固定                        | [Amazon Bedrock - Pin model versions](https://code.claude.com/docs/en/amazon-bedrock#4-pin-model-versions)  |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL`            | バックグラウンドタスク用モデルの固定          | 同上                                                                                                        |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 非推論通信の一括無効化                        | [Environment variables - Claude Code Docs](https://code.claude.com/docs/en/env-vars)                        |

:::details CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFICの詳細
このフラグは以下の個別フラグをまとめて有効化するオールインワンのトグルです[^7]。

- `DISABLE_AUTOUPDATER` — 自動アップデートの無効化
- `DISABLE_BUG_COMMAND` — バグ報告機能の無効化
- `DISABLE_ERROR_REPORTING` — エラー報告の無効化
- `DISABLE_TELEMETRY` — テレメトリの無効化

エアギャップ環境ではこれ1つで外部通信を最小化できます。

[^7]:
    `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`は`DISABLE_AUTOUPDATER`、`DISABLE_BUG_COMMAND`、`DISABLE_ERROR_REPORTING`、`DISABLE_TELEMETRY`を単一スイッチにまとめたもの — [Claude Code Settings Reference](https://claudefa.st/blog/guide/settings-reference)
    :::

### code-serverユーザー設定

```bash
CONFIG_DIR="${HOME}/.local/share/code-server/User"
mkdir -p "${CONFIG_DIR}"
cat > "${CONFIG_DIR}/settings.json" << 'EOF'
{
  "claudeCode.disableLoginPrompt": true,
  "claudeCode.selectedModel": "sonnet"
}
EOF
```

| 設定キー                        | 用途                                          | 出典                                                                                     |
| ------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `claudeCode.disableLoginPrompt` | Anthropicへのログインプロンプトを非表示にする | [Use Claude Code in VS Code - Claude Code Docs](https://code.claude.com/docs/en/vs-code) |
| `claudeCode.selectedModel`      | 使用モデルの選択                              | 同上（VS Code設定UIの`Extensions → Claude Code`セクション）                              |

## Step 5: 動作確認

```bash
# code-serverを再起動
sudo systemctl restart code-server@${USER}
```

code-serverのブラウザUIで以下を確認します。

1. **拡張機能の起動確認** — Claude Codeアイコン（Sparkアイコン）がサイドバーに表示されること
2. **プロバイダー確認** — チャットパネルで`/status`コマンドを実行し、**Provider: Amazon Bedrock** と表示されること
3. **推論テスト** — `Hello, are you working?` 等の簡単なプロンプトを送信し、レスポンスが返ること

## Step 6: トラブルシューティング

### 拡張機能が起動しない

CLIバイナリのダウンロードがブロックされている可能性が高いです。

```bash
# code-serverのログ確認
# Output Panel > "Claude Code" チャネルを確認
# または
journalctl -u code-server@${USER} --no-pager -n 50
```

**対策:** インターネットありの環境で一度起動し、CLIバイナリをベイクする（Step 3参照）。

### /status でBedrock表示にならない

環境変数が正しく反映されていません。

```bash
# code-serverプロセスの環境変数を確認
cat /proc/$(pgrep -f code-server)/environ | tr '\0' '\n' | grep CLAUDE

# ~/.claude/settings.json の内容を確認
cat ~/.claude/settings.json | python3 -m json.tool
```

### プロンプト送信後にタイムアウト

VPC Endpoint経路の問題です。

```bash
# VPC Endpoint疎通を再確認（Step 1）
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
nslookup bedrock-runtime.${REGION}.amazonaws.com

# Security Groupで443ポートが許可されているか確認
aws ec2 describe-security-groups \
  --query "SecurityGroups[*].[GroupId,IpPermissions[?ToPort==\`443\`]]" \
  --output table
```

チェックポイント:

- VPC Endpointに関連付けられたSecurity Groupでインバウンド443/TCPが許可されていること
- VPC Endpoint Policyで`bedrock:InvokeModel`等のアクションが許可されていること
- Private DNS が有効になっていること（VPC Endpoint設定で`Enable DNS name`がON）

### 認証エラー（403）

```bash
# Instance Profileの権限を確認
aws sts get-caller-identity
aws bedrock list-inference-profiles --region ${REGION} --max-results 1
```

- IAMポリシーが正しくアタッチされているか（Step 2参照）
- Bedrock Model CatalogでAnthropicのFTU（First Time Use）フォームを提出済みか

## まとめ

VPC閉域環境でClaude Code × Bedrockを動かすポイントは以下の3点です。

**1. 推論経路はVPC Endpoint（PrivateLink）で確保する**
`bedrock-runtime`、`bedrock`、`sts` の3つのVPC Endpointが必要です。

**2. 非推論の外部通信は`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`で一括抑止する**
自動アップデート、テレメトリ、エラー報告をまとめて無効化できます。

**3. 拡張機能とCLIバイナリは事前にオフラインで持ち込む**
エアギャップ環境最大のハマりポイントは初回起動時のCLIバイナリダウンロードです。検証環境で事前にバイナリを取得しAMIにベイクしておくことを推奨します。

これらが揃えば、Instance Profileによるキーレス認証でセキュアにClaude Codeを利用できます。

## 参考リンク

### Claude Code公式ドキュメント

- [Claude Code on Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock) — Bedrock統合の設定手順・IAMポリシー・モデル固定
- [Enterprise network configuration](https://code.claude.com/docs/en/network-config) — ネットワーク要件・プロキシ設定・必要なURL一覧
- [Claude Code Settings](https://code.claude.com/docs/en/settings) — settings.jsonの階層構造と設定キー一覧
- [Environment variables](https://code.claude.com/docs/en/env-vars) — 全環境変数リファレンス
- [Use Claude Code in VS Code](https://code.claude.com/docs/en/vs-code) — VS Code拡張機能の設定・Bedrockプロバイダー切り替え

### code-server

- [code-server FAQ](https://coder.com/docs/code-server/FAQ) — データディレクトリ・settings.jsonパス・拡張機能管理

### AWS

- [Amazon Bedrock VPC Endpoints (GitHub)](https://github.com/aws-samples/amazon-bedrock-vpc-endpoints) — VPC Endpoint構成サンプル
- [Guidance for Claude Code with Amazon Bedrock (GitHub)](https://github.com/aws-solutions-library-samples/guidance-for-claude-code-with-amazon-bedrock) — エンタープライズ向けデプロイパターン

### コミュニティ記事

- [Using Claude Code on AWS Bedrock (bespinian)](https://bespinian.io/en/blog/claude-code-using-aws-bedrock/) — Bedrock設定の実践ガイド・コスト見積もり
- [Configuring Claude Code Extension with AWS Bedrock (AWS in Plain English)](https://aws.plainenglish.io/configuring-claude-code-extension-with-aws-bedrock-and-how-you-can-avoid-my-mistakes-090dbed5215b) — 推論プロファイル設定のハマりポイント
- [Claude Code on Amazon Bedrock & AWS (Elevata)](https://elevata.io/en/claude-code-on-aws-complete-guide-bedrock-setup-self-hosted-models) — VPC/PrivateLink構成を含む包括的なデプロイガイド
