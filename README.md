# AWS用エージェントツールキット

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Build](https://github.com/aws/agent-toolkit-for-aws/actions/workflows/build.yml/badge.svg)](https://github.com/aws/agent-toolkit-for-aws/actions/workflows/build.yml)
[![Status](https://img.shields.io/badge/status-GA-green.svg)](https://github.com/aws/agent-toolkit-for-aws)

AIコーディングエージェントがAWS上でアプリケーションを構築、デプロイ、管理するのを支援します。

AWS用エージェントツールキットは、AIコーディングエージェントがAWSサービスを扱うために必要なツール、知識、ガードレール提供します。Claude Code、Codex、Cursor、Kiroなど、開発者がすでに使用しているコーディングエージェントで動作します。

## クイックスタート

### AWS CLI

AWS CLIを使用して、ターミナルからエージェントツールキットを直接使用します：

```
aws configure agent-toolkit
```

セットアップ、設定、使用方法については、[AWS CLI統合ガイド](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/aws-cli.html)を参照してください。

### Claude Code

プラグインは公式のAnthropicマーケットプレイス（`claude-plugins-official`）で利用可能です。これはClaudeコードのインストールにデフォルトで追加されます。
ツールキットからサポートされているプラグインをインストールするには、以下のコマンドを使用します。

サービス選択、CDK/CloudFormation、サーバーレス、コンテナ、ストレージ、可観測性、課金、SDK使用方法、デプロイメントをカバーする`aws-core`の場合：

```
/plugin install aws-core@claude-plugins-official
```

> **ヒント：** `Plugin not found`というエラーが出た場合は、まずローカルマーケットプレイスインデックスを更新してください：
>
> ```
> /plugin marketplace update claude-plugins-official
> ```

Amazon BedrockとAgentCoreを使用してAWS上でAIエージェントを構築するための`aws-agents`の場合：

```
/plugin install aws-agents@claude-plugins-official
```

S3テーブル、AWS Glue、Athenaを使用したデータレイク、分析、ETLワークフローをカバーする`aws-data-analytics`の場合：

```
/plugin install aws-data-analytics@claude-plugins-official
```

インシデント調査、コードレビュー、リリース準備のためのUAT実行、脆弱性スキャン、AWS DevOpsエージェントとAWSセキュリティエージェントを使用したペネトレーションテストを実行するための`aws-agents-for-devsecops`の場合：

```
/plugin marketplace add aws/agent-toolkit-for-aws
/plugin install aws-agents-for-devsecops
/reload-plugins

# またはClaudeの公式マーケットプレイスから：
/plugin install aws-agents-for-devsecops@claude-plugins-official
/reload-plugins

# セットアップ：
/aws-agents-for-devsecops:setup
```

### Codex

ターミナルで：

```
codex plugin marketplace add aws/agent-toolkit-for-aws
```

次にCodexを起動し、`/plugins`を実行して**aws-core**プラグインを参照してインストールします。

### Cursor

**設定 → プラグイン → チームマーケットプレイス → マーケットプレイスを追加 → リポジトリからインポート**からこのリポジトリをチームマーケットプレイスとして追加し、`aws/agent-toolkit-for-aws`をポイントします。Cursorはインポート時に[`.cursor-plugin/marketplace.json`](.cursor-plugin/marketplace.json)に列出されているプラグインをインデックスします。

次に**プラグイン**パネルを開き、**aws-core**プラグイン（ここから始めます）、または必要に応じて**aws-agents**と**aws-data-analytics**をインストールします。各プラグインはAWS MCPサーバー設定とエージェントスキルをバンドルしています。

### Kiro

Kiroのセットアップは2つの独立した部分で構成されています：AWS MCPサーバー（ランタイムAWS APIアクセスとドキュメント検索用）とローカルスキル（タスク固有のエージェントガイダンス用）。これらは相互に補完しますが、独立して動作します。スキルはMCPサーバーを必要とせず、MCPサーバーはローカルにインストールされたスキルを提供しません。

**1. AWS MCPサーバーを追加します**（Kiro MCP設定`.kiro/settings/mcp.json`に）：

```json
{
  "mcpServers": {
    "aws": {
      "command": "uvx",
      "args": [
        "mcp-proxy-for-aws@1.6.4",
        "https://aws-mcp.us-east-1.api.aws/mcp",
        "--metadata",
        "AWS_REGION=us-west-2"
      ]
    }
  }
}
```

> **注意：** サプライチェーンリスクから保護し、再現可能な動作を保証するために、特定のバージョン（例：`@1.6.4`）にピン留めすることをお勧めします。[PyPI](https://pypi.org/project/mcp-proxy-for-aws/)で定期的に新しい安定版をチェックし、それに応じて更新することをお勧めします。

MCPサーバーにより、エージェントはAWS API、サンドボックス化されたスクリプト実行、およびリアルタイムのドキュメント検索にアクセスできます。

**2. このリポジトリからスキルをインストールします**：

```
npx skills add aws/agent-toolkit-for-aws/skills
```

これにより、スキルファイルが`~/.kiro/skills/`（グローバル）または`.kiro/skills/`（プロジェクトレベル）にインストールされます。各スキルは`SKILL.md`ファイルと、オプションで必要に応じてエージェントがローカルファイルシステムから読み込む追加コンテキストが含まれた`references/`サブディレクトリを含むディレクトリです。Kiroはインストール済みスキルを自動的に検出し、タスクが一致するときにオンデマンドで有効化します。

> **前提条件：** [uv](https://docs.astral.sh/uv/)がインストールされている必要があります。AWS APIコールとスクリプト実行には、ローカルに設定されたAWSアカウント認証情報が必要ですが、ドキュメント検索またはスキル検出には不要です。詳細なセットアップ手順については、[ユーザーガイド](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/)を参照してください。

### その他のエージェント

エージェントでAWS MCPサーバーを設定する手順については、[AWS MCPサーバー入門ガイド](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/getting-started-aws-mcp-server.html)を参照してください。

次に、このリポジトリからスキルをインストールします：

```
npx skills add aws/agent-toolkit-for-aws/skills
```

> **前提条件：** [uv](https://docs.astral.sh/uv/)がインストールされている必要があります。AWS APIコールとスクリプト実行には、ローカルに設定されたAWSアカウント認証情報が必要ですが、ドキュメント検索またはスキル検出には不要です。詳細なセットアップ手順については、[ユーザーガイド](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/)を参照してください。

## 含まれるもの

### プラグイン

プラグインは、AWS MCPサーバー設定とエージェントスキルを単一のインストールにバンドルして、コーディングエージェント用に提供します。

| プラグイン                                                        | 説明                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [aws-core](plugins/aws-core/)                                 | AWSコアスキルとMCPサーバー設定。サービス選択、CDK/CloudFormation、サーバーレス、コンテナ、ストレージ、可観測性、課金、SDK使用方法、デプロイメントをカバーします。**ここから始めてください。**                                                                                                                                                                                    |
| [aws-agents](plugins/aws-agents/)                             | Amazon BedrockとAgentCoreを使用してAWS上でAIエージェントを構築するためのスキル。                                                                                                                                                                                                                                                                                           |
| [aws-data-analytics](plugins/aws-data-analytics/)             | S3テーブル、AWS Glue、Athenaを使用したデータレイク、分析、ETLワークフロー用のスキル。                                                                                                                                                                                                                                                                                           |
| [aws-agents-for-devsecops](plugins/aws-agents-for-devsecops/) | [AWS DevOpsエージェント](https://aws.amazon.com/devops-agent/?trk=7b4b0d25-1409-441c-b914-c5d08677c376&sc_channel=ghr)と[AWSセキュリティエージェント](https://aws.amazon.com/security-agent/?trk=7b4b0d25-1409-441c-b914-c5d08677c376&sc_channel=ghr)を使用して、インシデント調査、コードレビュー、リリース準備のためのUAT実行、脆弱性スキャン、ペネトレーションテストを実行します。 |

プラグインは現在Claude Code、Codex、Cursorで利用可能です。その他のエージェントについては、AWS MCPサーバーを直接設定し、このリポジトリからスキルをインストールしてください。

### スキル

エージェントスキルは、エージェントが特定のAWSタスクを完了するのを支援する、厳選された指示とリファレンスマテリアルのパッケージです。スキルはオンデマンドでロードされます。エージェントは現在のタスクに関連するもののみを検出して取得します。

```
npx skills add aws/agent-toolkit-for-aws/skills
```

すべての利用可能なスキルを確認するには、[`skills/`](skills/)ディレクトリを参照してください。

### ルールファイル

推奨されるプロジェクトレベルの設定ファイル。これらはエージェントにAWSの最も効果的な使用方法を指示します。例えば、AWS MCPサーバーの使用、利用可能なスキルの検出、アクション前のドキュメント検索などです。

詳細については[`rules/`](rules/)を参照してください。

### AWS MCPサーバー

[AWS MCPサーバー](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/understanding-mcp-server-tools.html)は、エージェントにモデルコンテキストプロトコルを通じてAWSへのアクセスを提供するマネージドサーバーです。以下を提供します：

- **フルAWS APIカバレッジ** — 単一の認証エンドポイントを通じて300以上のAWSサービスのいずれかと相互作用します。
- **サンドボックス化されたスクリプト実行** — エージェントは複雑なマルチステップ操作のための分離された環境でPythonスクリプトを実行できます。
- **リアルタイムドキュメントアクセス** — 認証なしで、現在のAWSドキュメント、APIリファレンス、サービス機能を検索して取得します。
- **エンタープライズコントロール** — Amazon CloudWatchメトリクス、エージェント固有ポリシー用のIAMコンテキストキー、AWS CloudTrail監査ログ。

操作、利用可能なツール、認証、サポートされるリージョンの詳細については、[AWS MCPサーバーのドキュメント](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/understanding-mcp-server-tools.html)を参照してください。

## ドキュメント

- [ユーザーガイド](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/) — セットアップ、設定、リファレンスドキュメント。
- [AWS MCPサーバーツール](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/understanding-mcp-server-tools.html) — 利用可能なすべてのMCPツールのリファレンス。

## エージェントツールキットとAWS Labsのサーバー、スキル、プラグインとの関係

2025年、AWSは[AWS Labs](https://github.com/awslabs)の一部としてMCPサーバー、スキル、プラグインのリリースを開始しました。AWS用エージェントツールキットはこれらのツールの後継です。以下を含む主な機能を提供しているため、AWS用エージェントツールキットの使用をお勧めします：

- エージェントアクションとヒューマンアクションを区別するIAM条件キー。これにより、エージェント専用のポリシーを記述できます。たとえば、ユーザーの基になるIAMロールが書き込みアクション実行できる場合でも、MCPサーバーを通じた読み取り専用アクションのみを許可するポリシーを記述できます。
- すべてのリクエストのCloudWatchメトリクスとCloudTrail監査ログ。これにより、コーディングエージェント活動を監視および監査できます。
- 徹底的なエンドツーエンド評価を受けたエージェントスキル。ワークフローが正常に完了することを確認できます。

[AWS Labs](https://github.com/awslabs)のMCPサーバー、スキル、プラグインは引き続き機能し、貢献を受け付けます。時間とともに、AWS Labsの最高のものはAWS用エージェントツールキットに移行され、お客様が最も広いツールセットと行うための確実性を得ることができます。

## ライセンス

このプロジェクトはApache-2.0ライセンスの下でライセンスされています。詳細は[LICENSE](LICENSE)を参照してください。
