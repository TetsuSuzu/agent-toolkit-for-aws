---
name: agents-build
description: >
  既存のエージェントプロジェクトを拡張するときに使います。対象は、Memory、
  アプリ統合、VPC、マルチエージェント、移行、モデル、ブラウザ、Code Interpreter、
  Payments、リソースの削除です。トリガー: "add memory"、
  "remember across sessions"、"call agent from app"、"invoke agent from
  code"、"agent auth"、"streaming"、"VPC"、"VPC
  connectivity"、"can't reach from VPC"、"multi-agent"、
  "A2A"、"A2A auth"、"orchestrator not delegating"、"specialist not
  called"、"migrate Bedrock Agent"、"migration issue"、"change model"、
  "browser tool"、"code
  interpreter"、"delete agent"、"tear down"、"agentcore remove"、
  "cross-account memory"、
  "add payments capability to my agent"、"wire payments plugin"、
  "integrate x402 payments with the agent I'm building"、
  "add MPP payments"、"Machine Payments Protocol"。
  Gateway 経由の外部 API: agents-connect を使います。新規プロジェクト:
  agents-get-started を使います。CLI/開発サーバーのエラー: agents-debug を使います。
  実行時の x402/MPP 支払い: agents-pay を使います。移行に関する Strands と
  LangGraph の選択は、ここで扱います。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# build

AgentCore のエージェントプロジェクトに機能を追加します。

## 使うタイミング

- エージェントに、セッションをまたぐ Memory を追加する
- デプロイ済みのエージェントを、Web アプリ、モバイルアプリ、バックエンドサービスから呼び出す
- プライベートなリソース（RDS、内部 API）向けに VPC ネットワークを設定する
- オーケストレーター/スペシャリストのパターンで、マルチエージェントのシステムを構築する
- 既存の Bedrock Agent を AgentCore に移行する
- エージェントが Web サイトを操作できるように、Browser ツールを追加する
- エージェントがサンドボックスでコードを実行できるように、Code Interpreter を追加する
- エージェントが x402 または MPP で保護された API、ツール、コンテンツの料金を支払えるように、AgentCore Payments を追加する
- プロジェクトからリソースを削除する、またはデプロイを撤去する

次の場合は使わないでください:

- Gateway を通じた外部のツール/API への接続（OpenAPI 仕様、Lambda、MCP サーバー、認証情報、ポリシー）→ `agents-connect` を使います
- 新規プロジェクトのスキャフォールディング → `agents-get-started` を使います
- デプロイ → `agents-deploy` を使います

## 入力

`$ARGUMENTS` には次のものを指定できます。

- 機能: "memory"、"integrate"、"vpc"、"multi-agent"、"migrate"、"browser"、"code-interpreter"、"payments"、"teardown"
- やりたいことの説明: "remember user preferences"、"call from React app"、"scrape a website"、"run pandas in the agent"、"delete my agent"、"clean up resources"
- 空 — スキルがコンテキストからワークフローを判断します

## 手順

### ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。

古い場合: 「`agentcore update` を実行して、最新のバージョンにしてください。」

### ステップ 1: プロジェクトのコンテキストを読み取る

`agentcore/agentcore.json` を読み、現在のプロジェクト（フレームワーク、既存のリソース、エージェントの設定）を把握します。

`agentcore/agentcore.json` が見つからない場合:

1. **開発者が、間違ったディレクトリにいないか確認します。** 親ディレクトリ（最大 3 階層まで）で `agentcore/agentcore.json` を探します。見つかった場合は、次のように伝えます。「`<path>` に AgentCore のプロジェクトが見つかりました。そのプロジェクトで作業していますか?」
2. **近くにプロジェクトがまったく存在しない場合**は、どの機能を追加したかったのかを質問します。そのうえで、2 つの進め方を提案します。
   - 「まずプロジェクトを作成してから、CAPABILITY を追加する流れを案内できます。そうしますか?」（get-started のフローをその場で実行し、続けてビルドのワークフローに進む）
   - 「すでに別の場所にプロジェクトがある場合は、そこに `cd` してからやり直してください。」

「agents-get-started を使ってください」と言うだけで終わらせないでください。開発者が実際にやりたかったことの文脈が失われます。

### ステップ 2: ワークフローを判断する

**重要な切り分け** — ビルド用のリファレンスに振り分ける前に、そのプロンプトが、実際には connect や debug の関心事ではないか確認します。

- 外部 API、Lambda 関数、OpenAPI 仕様、Gateway、認証情報、MCP サーバー、ポリシーに言及している場合 → これは build ではなく `agents-connect` です
- 開発者が、何かが壊れていると言っている場合（誤った回答、エラー、ツールの失敗）→ これは build ではなく `agents-debug` です
- build は、動作しているプロジェクトに**新しい機能を追加する**ためのもので、壊れたものを直すためのものではありません

開発者のプロンプトと `$ARGUMENTS` に基づいて、適切なリファレンスを読み込みます。

| 開発者の意図 | 読み込むリファレンス |
|---|---|
| Memory の追加、物事を記憶する、ユーザーの好み、セッションをまたぐ | [`references-ja/memory.md`](references-ja/memory.md) |
| アプリからエージェントを呼び出す、コードから呼び出す、ストリーミング、SDK クライアント、エージェントの URL、セッション内でシェルを実行する | [`references-ja/integrate.md`](references-ja/integrate.md) |
| VPC、プライベートネットワーク、RDS、内部 API、サブネット、セキュリティグループ | [`references-ja/vpc.md`](references-ja/vpc.md) |
| マルチエージェント、オーケストレーター、スペシャリスト、A2A、委任、エージェントの引き継ぎ | [`references-ja/multi-agent.md`](references-ja/multi-agent.md) |
| 呼び出し元からエージェントへのカスタムヘッダー、ヘッダーの許可リスト、テナント ID/相関 ID/トレースの伝播 | [`references-ja/request-headers.md`](references-ja/request-headers.md) |
| Bedrock Agent の移行、エージェントのインポート、AgentCore への移行 | [`references-ja/migrate.md`](references-ja/migrate.md) |
| Browser ツール、Web ナビゲーション、フォーム入力、スクレイピング、Nova Act、Playwright、ライブビュー | [`references-ja/browser.md`](references-ja/browser.md) |
| Code Interpreter、コードの実行、サンドボックス、Python/JS/TS の実行、エージェント内でのデータ分析、pandas | [`references-ja/code-interpreter.md`](references-ja/code-interpreter.md) |
| Payments、x402 または MPP のコンテンツへの支払い、402 Payment Required、Machine Payments Protocol、WWW-Authenticate: Payment、マイクロトランザクション、有料の API/ツール、payment manager/connector | [`references-ja/payments.md`](references-ja/payments.md) |
| エージェントの削除、リソースの削除、撤去、クリーンアップ、破棄、最初からやり直す | [`references-ja/teardown.md`](references-ja/teardown.md) |
| モデルの変更、モデルの切り替え、Haiku/Sonnet/Nova の使用、別のモデル | インライン — 下の「モデルを変更する」を参照 |

開発者がローカル開発とデプロイ済み環境の違いについて質問した場合（例: 「デプロイ後は Memory が動くのに、ローカルでは動かないのはなぜ?」）は、個別のワークフローのリファレンスと併せて、[`references-ja/local-vs-deployed.md`](references-ja/local-vs-deployed.md) を読み込みます。

一致するファイルをコンテキストに読み込み、その「手順」のセクションに順番に従ってください。要約はしないでください。

意図が曖昧な場合は、どの機能を追加したいのかを開発者に尋ねます。

### モデルを変更する

モデルは、`app/<AgentName>/model/load.py`（`agentcore create` が生成します）で設定します。変更するには:

1. `app/<AgentName>/model/load.py` を開く
2. `BedrockModel()` コンストラクターの `model_id` パラメーターを変更する

```python
# デフォルト（CLI が生成したもの）
return BedrockModel(model_id="global.anthropic.claude-sonnet-4-5-20250929-v1:0")

# コスト削減のために Haiku に切り替える
return BedrockModel(model_id="us.anthropic.claude-3-5-haiku-20241022-v1:0")

# Nova Lite に切り替える
return BedrockModel(model_id="amazon.nova-lite-v1:0")
```

クロスリージョン推論プロファイルのプレフィックス（`us.`、`eu.`、`apac.`、`global.`）は、推論が実行される場所を制御します。最大のスループットが欲しい場合は `global.` を、データレジデンシーが必要な場合は地域のプレフィックスを使います。すべてのモデルがすべてのプレフィックスをサポートしているわけではありません。Bedrock の推論プロファイルのドキュメントで確認してください。

モデルを変更したあと:

- そのモデルがリージョンで有効になっていることを確認する: AWS コンソール → Amazon Bedrock → Model access
- クロスリージョンプロファイルの場合は、すべての宛先リージョンで有効にする
- `agents-harden` を使っている場合は、IAM ポリシーを更新して、新しいモデルの ARN に絞り込む
- `agentcore dev` でローカルテストしてから、`agentcore deploy` でデプロイ済みのエージェントを更新する

`agentcore.json` を変更する必要はありません。モデルは、プロジェクトの設定ではなく、コードで設定されます。

### 事前確認: CLI コマンドを生成する前に `--name` を検証する

どのリファレンスを読み込んだ場合でも、ほとんどは `agentcore add <resource> --name <something>` というコマンドを生成することになります。CLI は、無効な名前に対して**遅れて**失敗します。プロンプトを一通り進めたあとにエラーが出て、コマンドを実行する前には分かりません。事前に検証してください。

| リソース | 最大文字数 | 使用できる文字 | 先頭の文字 |
|---|---|---|---|
| エージェント（`add agent`） | 48 | 英数字 + `_` | 英字 |
| Memory、gateway、gateway-target、credential、evaluator、online-eval、policy、policy-engine、payment-manager、payment-connector | 48 | 英数字 + `_` | 英字 |

コマンドを組み立てる前に、文字数を数えてください。名前が上限を超えているか、ハイフン、ドット、スペースが含まれている場合は、押し戻します。「`<name>` は N 文字です / `-` を使っていて、CLI は受け付けません。`<suggestion>` はどうでしょうか?」無効な名前のままコマンドを実行して、CLI のメッセージが分かりやすいことを期待しないでください。

注意: `agentcore create --name`（プロジェクト名）には、**より厳しい 23 文字の上限**があり、アンダースコアも使えません。これは `agents-get-started` で扱っています。開発者が create をやり直している場合は、特に 23 文字の上限を指摘してください。

## 出力

ワークフローによって異なります。具体的な出力は、読み込んだリファレンスを参照してください。

## 品質基準

- 開発者の意図に基づいて、正しいリファレンスが読み込まれている
- すべての出力が、読み込んだリファレンスの品質基準に従っている
- 関連する場合は、他のスキル（agents-connect、agents-deploy）への相互参照が含まれている
