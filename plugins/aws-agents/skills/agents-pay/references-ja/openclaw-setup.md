# OpenClaw のセットアップ

OpenClaw は、ClawHub の `@aws/aws-agents-pay` パッケージを使います。その正式なソースは、このリポジトリの `plugins/aws-agents/skills/agents-pay/packages/openclaw/` にあります。

AgentCore Payments の背景については、[getting started guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-getting-started.html) を参照してください。

## ステップ 0: ユーザーのアイデンティティを選ぶ

支払いリソースをプロビジョニングする前に、安定した `userId` を選びます。このアイデンティティは、`create-instrument`、`new-session`、OpenClaw プラグインの設定で一貫して使う必要があります。不一致だと、セッションは instrument を支出できません。

```bash
export PAYMENT_USER_ID="my-openclaw-agent"
```

すべての管理コマンドに `--user-id $PAYMENT_USER_ID` を渡すか、先に `init-config` を実行してください(`~/.agents-pay/config.json` に、1 つ生成して保存します)。

## インストール

```bash
openclaw plugins install clawhub:@aws/aws-agents-pay
```

プラグインは、支払いリソースが設定される前にも読み込めるため、同梱のセットアップ用スキルは見つけられるままです。完全で信頼できる設定が存在するまでは、2 つの支払いツールはどちらも、安全側に倒れて失敗します。

## AgentCore プロジェクトをプロビジョニングする

支払いリソースを追加してデプロイする前に、AgentCore のプロジェクトを作成します。

```bash
npm install -g @aws/agentcore
PROJECT_NAME=openclaw-payments
agentcore create --project-name "$PROJECT_NAME" --no-agent
cd "$PROJECT_NAME"
agentcore add payment-manager
agentcore add payment-connector
agentcore deploy
export AGENTCORE_PROJECT_DIR="$PWD"
```

認証情報が対話式のターミナルの中にとどまるように、両方の `agentcore add` コマンドをフラグなしで実行してください。プラグインのディレクトリからセットアップのウィザードを実行する間は、`AGENTCORE_PROJECT_DIR` を設定したままにしてください。

## クイックセットアップ(推奨)

AgentCore のプロジェクトをデプロイしたら、対話式のウィザードを実行します。支払い instrument とセッションを作成し、OpenClaw の設定を生成します。

```bash
cd ~/.openclaw/extensions/aws-agents-pay/skills/agents-pay
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python scripts/agents_pay_admin.py setup-openclaw \
  --project-dir "$AGENTCORE_PROJECT_DIR"
```

ウィザードは、次の順に案内します: ユーザーのアイデンティティ → ネットワーク → 受取人 → 支出の上限 → instrument の作成 → 委任/入金 → セッションの承認 → そのまま貼り付けられる OpenClaw の設定 JSON を生成。

## 手動セットアップ

OpenClaw の設定(`~/.openclaw/openclaw.json`、または `openclaw config` 経由)に追加します。インストールしたプラグインを明示的に許可してください。

支払いのインフラやセッションが存在しない場合は、LLM のワークフローを止めてください。別のターミナルを開き、同梱のスキルから、人間によるセットアップを実行します。

```bash
cd ~/.openclaw/extensions/aws-agents-pay/skills/agents-pay
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python scripts/agents_pay_admin.py init-config \
  --max-per-payment-usd 0.05 \
  --network eip155:84532 \
  --recipient 0xMerchantWalletAddress
python scripts/agents_pay_admin.py create-instrument --email you@example.com
python scripts/agents_pay_admin.py new-session \
  --budget 1.00 \
  --expiry-minutes 60
```

2 つの `agentcore add` コマンドは、認証情報が対話式のターミナルの中にとどまるように、フラグなしで実行する必要があります。認証情報、コマンドの出力、デプロイの状態、生成された識別子を、LLM との会話に貼り付けないでください。

セッションを作成する前に、委任とテストネットの入金を完了してください。その後、同じターミナル、またはローカルのエディターで、OpenClaw の設定をローカルで編集します。

> **重要:** 以下の `userId` は、`create-instrument` と `new-session` で使ったアイデンティティと一致する必要があります。不一致だと、セッションは instrument を支出できません。

```json
{
  "plugins": {
    "allow": ["aws-agents-pay"],
    "entries": {
      "aws-agents-pay": {
        "enabled": true,
        "config": {
          "region": "us-east-1",
          "paymentManagerArn": "arn:aws:bedrock-agentcore:us-east-1:ACCOUNT:payment-manager/NAME",
          "paymentInstrumentId": "payment-instrument-XXXX",
          "payment_session_id": "payment-session-XXXX",
          "userId": "your-user-id",
          "networkPreferences": ["eip155:84532"],
          "allowedOrigins": ["https://merchant.example"],
          "allowedRecipients": ["0xMerchantWalletAddress"],
          "allowedAssetsByNetwork": {
            "eip155:84532": [
              "0x036CbD53842c5426634e7929541eC2318f3dCF7e"
            ]
          },
          "maxPaymentAmountAtomic": "100000"
        }
      }
    }
  }
}
```

`paymentManagerArn`、`paymentInstrumentId`、`payment_session_id`、`userId` は、人間が実行する管理のセットアップから得られます。オリジン、受取人、ネットワーク、正確なアセットのコントラクトは、帯域外で承認された値で設定してください。発行元が選択する受取人を意図的に許可する場合は、`allowedRecipients` を `"allowAnyRecipient": true` に置き換えます。プラグインは、両方のフィールドを同時に指定すると拒否します。`maxPaymentAmountAtomic` の例の値は `100000`(6 桁の小数で 0.10 USDC)で、デフォルトは提供されていません。

## 仕組み

OpenClaw は、この TypeScript プラグインの `get_paid_content` を使って、支払いを実行します。ポリシーの検証、SSRF の制御、冪等性、証明の分離、加盟店へのリプレイは、TypeScript の側に残ります。プラグインは、シェルを使わずに、パッケージ相対の固定された Python のヘルパーを呼び出します。呼び出すのは、`GetPaymentSession` と `ProcessPayment` のためだけです。上で作成したパッケージ内の `.venv` が、boto3 と標準の AWS 認証情報チェーンを提供します。Python の `x402_fetch` ランタイムは呼び出しません。その他のエージェントホストでは、代わりに `SKILL.md` に記載された Python のランタイムを使い、両方の経路を同時に有効にしないでください。

プラグインが公開するのは、範囲を限定した 2 つの OpenClaw ランタイムツールだけです。

| ツール | 動作 | 権限 |
|---|---|---|
| `get_payment_session_status` | セッションの予算/有効期限を確認する | 読み取り専用 |
| `get_paid_content` | 承認された x402 の URL に支払い、レスポンスのメタデータのみを返す | 支出(事前に承認されたセッションの予算の範囲内) |

### セキュリティの境界

- **一般的なシェルアクセスなし。** エージェントは、コマンド、スクリプト、パス、環境変数を選べません。プラグインが起動するのは、固定された AgentCore のヘルパーだけで、`shell: false`、サイズが制限された JSON、タイムアウト、出力の制限が付いています。
- **実行時にセットアップしない。** Payment Manager、connector、認証情報プロバイダー、instrument、セッションの作成は、モデルから見えるランタイムの外で行われます。
- **認証情報を露出しない。** ウォレットプロバイダーのシークレットが、ツールのインターフェースに現れることはありません。以前にチャットから入力した、プロバイダーの認証情報は、ローテーションしてください。
- **代わりのセッションなし。** ランタイムは、新しいセッションを発行できません。設定したセッションが期限切れになったり、使い果たされたりした場合は、信頼できる管理の経路で新しいものを作成し、設定を更新します。
- **署名済みの証明を露出しない。** 署名された支払いの証明は、信頼できるリクエスト処理のコードの内部でのみ付与され、ツールの出力として返されることはありません。
- **信頼できるチャレンジのポリシー。** ランタイムのコードは、`accepts[0]` を信頼する代わりに、承認されたオファーを選択し、`ProcessPayment` の前に、HTTPS、公開の宛先、リダイレクトなし、設定されたオリジンとネットワーク、正確なアセット、正で上限のある金額、承認された受取人、一致するリソースを強制します。
- **x402 v2 のみ。** 支払い後のリプレイは、必須の `PAYMENT-SIGNATURE` ヘッダーを使います。v1 のチャレンジは、安全側に倒れて失敗します。
- **安定したリトライ。** AgentCore のクライアントトークンは、セッション、リソース、ネットワーク、アセット、受取人、金額を結び付け、発行元が制御するノンスを除外します。
- **発行元のコンテンツの分離。** 支払い済みのレスポンスの本文は、支払いが可能なモデルのコンテキストに返されません。`get_paid_content` が返すのは、ステータス、コンテンツタイプ、バイト数、SHA-256 のハッシュだけです。
- **予算の強制。** AgentCore のサービスは、エージェントが何を要求するかに関係なく、`maxSpendAmount` と `expiryTimeInMinutes` を、サーバー側で強制します。

## 更新を公開する

ビルドと公開のワークフローについては、ソースのリポジトリの `plugins/aws-agents/skills/agents-pay/packages/openclaw/PUBLISHING.md` を参照してください。
