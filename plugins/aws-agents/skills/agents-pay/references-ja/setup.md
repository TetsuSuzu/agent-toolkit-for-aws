# agents-pay のセットアップ

プロビジョニングの完全なウォークスルーです。最後の組み込みを除き、**ここでのすべてのステップは人間が実行します。** エージェントが呼び出せるものについては、SKILL.md のツールの一覧を参照してください。

## 0. インストール

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
npm install -g @aws/agentcore
agentcore --version
python -c "from bedrock_agentcore.payments import PaymentManager; print('payments OK')"
```

このインポートが失敗する場合は、先に進む前に、Payments をサポートする `bedrock-agentcore` のリリースをインストールしてください。以下の Python コマンドのために、仮想環境を有効にしたままにします。

`requirements.txt` は、このスキルが使う Python のランタイムの依存関係グラフ全体を固定しています。自分のリリース作業で整合性を固定するには、その正確なグラフから、プラットフォーム固有のハッシュを生成します。

```bash
python -m pip download -r requirements.txt -d wheels/
python -m pip hash wheels/*
```

## 1. プロバイダーの認証情報を取得する

connector のコマンドを実行する前に、これを行ってください。**これらを、チャット、エージェントのプロンプト、コマンドのフラグに、決して貼り付けないでください。**

**Coinbase CDP** — <https://portal.cdp.coinbase.com/>

1. CDP のプロジェクトを作成する、または開く
2. API キーを作成する — **API Key ID** と **API Key Secret** を控える
3. **Wallet Secret**(署名操作に使用)を生成する
4. Project → Wallet → Embedded Wallets → Policies → **Delegated signing を有効にする**(必須)

**Stripe Privy** — <https://dashboard.privy.io/>

1. AgentCore 専用の Privy アプリを**新規に**作成する(既存のアプリを再利用しない)
2. **App ID** と **App Secret** をコピーする
3. Wallet Infrastructure → Authorization → New Key → P-256 の鍵ペアを生成する
4. 秘密鍵から `wallet-auth:` プレフィックスを削除する。生の base64(`MIGHAgEA...` で始まるもの)を残す
5. **Authorization ID** を控える

## 2. payment manager を作成する

LLM との会話から離れて、別のターミナルでこれを実行してください。認証情報、コマンドの出力、デプロイの状態、生成された ID を、チャットに貼り付けないでください。セットアップ全体を対話式のターミナルの中に保つために、フラグなしのコマンドを使います。

```bash
agentcore add payment-manager
```

名前: 英字で始まり、英数字とアンダースコア、48 文字以内。

任意で、サービスがスキルでオンボードされたリソースを区別できるように、プロジェクトにタグを付けます — `agentcore/agentcore.json` の最上位の `tags` オブジェクトに、既存のエントリを残したまま追加します。

```json
"tags": { "agentcore:onboarding-source": "agent-toolkit-skill" }
```

## 3. payment connector を作成する

**フラグなしで**実行します。ウィザードがシークレットごとに入力を求めるので、シェルの履歴やプロセス一覧には何も残りません。

```bash
agentcore add payment-connector
```

`--manager`、`--name`、`--provider` を渡すと、CLI は非対話モードに切り替わり、その後は*すべてのシークレットをフラグで渡すことが必須*になります。入力を求める動作には戻りません。フラグなしのコマンドを使ってください。

CLI は、プロバイダーのシークレットを `agentcore/.env.local` に**平文で**書き込み、デプロイ時に AgentCore Identity にアップロードします。デプロイする前に:

```bash
grep -q '^\.env\.local$\|^\.env\.\*$' .gitignore || echo '.env.local' >> .gitignore
chmod 600 agentcore/.env.local
```

Python のスキャフォールドのデフォルトの `.gitignore` は `.env` しか列挙していないため、`.env.local` を追加する必要があります。**エージェントは、このファイルを決して読んではなりません。**

## 4. デプロイする

```bash
agentcore deploy
```

manager と connector、そして `Payment<Name>ProcessPaymentRole` をプロビジョニングします。ID は、`agentcore/.cli/deployed-state.json` の `targets.<target>.resources.payments[]` の下(`managerArn`、`connectors[].connectorId`、`processPaymentRoleArn`)に書き込まれます。

## 5. IAM: 4 つのロールのモデル

**公式のガイドに従ってください:**
<https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-iam-roles.html>

AgentCore Payments は、4 つのロールを定義しています。このスキルの設計はそれを前提としており、以下のセキュリティ上の性質は、ご利用のアカウントで分離が実際に機能していることに依存します。

| ロール | 目的 | 引き受ける主体 |
|---|---|---|
| **ControlPlaneRole** | payment manager、connector、認証情報プロバイダー | 管理者 |
| **ManagementRole** | 支払い**instrument とセッション**。`ProcessPayment` は明示的に拒否される | この管理用の CLI を実行する人間 |
| **ProcessPaymentRole** | 支払いを実行する。セッションを作成できない | エージェントのランタイム |
| **ResourceRetrievalRole** | 認証情報を取得するために AgentCore が引き受けるサービスロール | `bedrock-agentcore.amazonaws.com` |

AWS のガイドは、その理由を直接述べています。

> 「支払いの管理と支払いの実行を分離することで、単一の侵害された ID が、無制限の予算でセッションを作成することと、そのセッションに対して支払いを実行することの両方を行うのを防ぎます。」

この分離により、支払いの管理と実行が、異なる信頼されたロールに保たれます。

### どのロールがどのコマンドを実行するか

| ステップ | コマンド | ロール |
|---|---|---|
| 1 | `agentcore add payment-manager` / `payment-connector` / `deploy` | **ControlPlaneRole** |
| 2 | `agents_pay_admin.py init-config` | なし — ローカルファイルを書き込むだけ |
| 3 | `agents_pay_admin.py create-instrument` | **ManagementRole** |
| 4 | `agents_pay_admin.py new-session` | **ManagementRole** |
| 6 | エージェントが `x402_fetch` を呼び出す | **ProcessPaymentRole** |

### ManagementRole — 管理用の CLI を実行する人間向け

明示的な `Deny` に注目してください。これは飾りではありません。このロールが、予算の発行と支出の両方に使えてしまうことを防ぐものです。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPaymentManagement",
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:CreatePaymentInstrument",
        "bedrock-agentcore:GetPaymentInstrument",
        "bedrock-agentcore:ListPaymentInstruments",
        "bedrock-agentcore:DeletePaymentInstrument",
        "bedrock-agentcore:CreatePaymentSession",
        "bedrock-agentcore:GetPaymentSession",
        "bedrock-agentcore:ListPaymentSessions",
        "bedrock-agentcore:DeletePaymentSession"
      ],
      "Resource": [
        "arn:aws:bedrock-agentcore:*:ACCOUNT:payment-manager/*/instrument/*",
        "arn:aws:bedrock-agentcore:*:ACCOUNT:payment-manager/*/session/*"
      ]
    },
    {
      "Sid": "DenyProcessPayment",
      "Effect": "Deny",
      "Action": "bedrock-agentcore:ProcessPayment",
      "Resource": "*"
    }
  ]
}
```

### ProcessPaymentRole — エージェントのランタイム向け

`ProcessPayment` と読み取りです。**セッションの書き込みアクションはありません。**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowProcessPayment",
      "Effect": "Allow",
      "Action": "bedrock-agentcore:ProcessPayment",
      "Resource": ["arn:aws:bedrock-agentcore:*:ACCOUNT:payment-manager/*/session/*"]
    },
    {
      "Sid": "AllowPaymentReadOperations",
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:GetPaymentInstrument",
        "bedrock-agentcore:GetPaymentInstrumentBalance",
        "bedrock-agentcore:GetPaymentSession"
      ],
      "Resource": [
        "arn:aws:bedrock-agentcore:*:ACCOUNT:payment-manager/*/instrument/*",
        "arn:aws:bedrock-agentcore:*:ACCOUNT:payment-manager/*/session/*"
      ]
    }
  ]
}
```

AWS のガイドは、避けるべき組み合わせについて強調しています。

> 「PaymentSession の*書き込み*権限(たとえば `CreatePaymentSession`)と `ProcessPayment` を同じロールに含めないでください。そうしないと、呼び出し元は、より大きな予算で新しいセッションを作成することで、支払いの上限を回避できます。」

デプロイ後に確認します。

```bash
aws iam get-role-policy --role-name <ProcessPaymentRole> --policy-name <Policy> \
  | grep -c CreatePaymentSession        # 0 であること
```

ランタイムで `CreatePaymentSession` に対する `AccessDeniedException` が出るのは、**制御が機能している証拠**です。権限を付与して「修正」しないでください。

### ResourceRetrievalRole

AgentCore が実行時に引き受けるサービスロールです。`agentcore deploy` によって作成、接続されます。人間には割り当てられません。信頼ポリシーは、`aws:SourceAccount` の条件を付けて、`bedrock-agentcore.amazonaws.com` を指定する必要があります。ベースと、connector ごとの権限の詳細は、ガイドを参照してください。

## 6. 支払いポリシーを書く

```bash
python3 scripts/agents_pay_admin.py init-config \
  --max-per-payment-usd 0.05 \
  --network eip155:84532 \
  --recipient 0xMerchantWalletAddress
```

許可リストに追加する前に、加盟店の実際の `payTo` と金額を調べるには、支払わずにそのチャレンジを読み取ります。

```bash
curl -sD - https://<merchant-host>/<path> -o /dev/null \
  | awk '/^payment-required:/{print $2}' | base64 -d | python3 -m json.tool
```

`accepts[].payTo`、`.amount`(整数の基本単位。USDC は小数点以下 6 桁なので、`2000` = $0.002)、`.asset`、`.network` を確認し、`--recipient` で慎重に許可リストに追加します。未知の受取人は、署名の前に拒否されます。

オペレーターが、発行元が選択する受取人を意図的に受け入れる場合は、すべての `--recipient` フラグを `--allow-any-recipient` に置き換えます。これらのモードは排他的です。このフラグは、スキーム、ネットワーク、アセット、オリジン/リソース、1 回あたり、セッションの累積の各上限を、緩和しません。

エージェントを既知の加盟店の集合に固定したい場合に限り、`--origin https://host`(繰り返し指定可能)を追加します。省略すると、公開されている任意の HTTPS サイトを取得できます。

## 7. ユーザー単位の instrument を作成する

このインストールに 1 つのウォレットです。支払者のアイデンティティは、`init-config` が生成した設定から読み取られます。このスキルはシングルテナントなので、考え出したり、同期を保ったりするものはありません。

```bash
python3 scripts/agents_pay_admin.py create-instrument --email you@example.com
```

manager の ARN と connector の ID は、`agentcore/.cli/deployed-state.json` から自動で読み取られるため、AgentCore のプロジェクトのディレクトリから実行してください。そうでない場合は、`--manager-arn` と `--connector-id` を明示的に渡します。

Solana の場合は `--network-family SOLANA` を追加します。デフォルトの `ETHEREUM` は、Base と Base Sepolia をカバーします。

ランタイムのロールではなく、**セットアップ用の**ロールで実行してください。instrument ID、ウォレットアドレス、委任の URL、ランタイム用の `export` 行が表示されます。

## 8. 委任と入金

ウォレットごとに一度だけ行います。**セッションを作成する前に、これを行ってください。**

どちらのステップも、セッションではなく*ウォレット*に関連付けられます。`create_payment_session` が受け取るのは、ユーザー、有効期限、予算だけです。セッションには時間制限があります(デフォルトは 60 分)。そのため、先にセッションを作成してから、ブラウザのフローとフォーセットからの送金を完了しに行くと、時間が無駄に過ぎるだけで、エージェントが何も支出しないうちに予算が期限切れになることがあります。

**委任** — エージェントがウォレットから支払うことを承認します。

- *Coinbase CDP*: エンドユーザーが `redirectUrl` にアクセスし、サインインして、ウォレットアドレスに権限を付与します
- *Stripe Privy*: リダイレクト URL はありません。Privy のフロントエンド SDK(<https://github.com/privy-io/aws-agentcore-sdk>)を使い、エンドユーザーのメールアドレスでサインインして、委任を承認します

**入金** — <https://faucet.circle.com/>(Base Sepolia)から、ウォレットアドレスにテストネットの USDC を送ります。エージェントが支出する可能性のある金額だけを入金してください。他のすべての制御が失敗した場合に備えて、ウォレットの残高が最後の砦になります。

## 9. セッションを承認して、preflight を実行する

```bash
python3 scripts/agents_pay_admin.py new-session --budget 1.00 --expiry-minutes 60

export PAYMENT_MANAGER_ARN=...   PAYMENT_INSTRUMENT_ID=...
export PAYMENT_SESSION_ID=...    PAYMENT_USER_ID=alice
export AWS_REGION=us-west-2

python3 scripts/agents_pay_admin.py preflight
python3 scripts/test_x402_policy.py
```

セッションは短く(60 分以内)、予算は小さく保ってください。予算を使い切ったら、人間が再び `new-session` を実行します。エージェントにはできません。

## ネットワーク

2 つの異なる概念があり、これを混同するのは、セットアップ時のよくある失敗です。

- **ネットワークファミリー** — instrument を作成するときに使う(`--network-family`)
- **チェーン** — x402 のチャレンジと、このスキルの `allowed_networks` に現れる、CAIP-2 の識別子

**ファミリー(instrument の作成):**

| ファミリー | 値 | カバーするもの | プロバイダー |
|---|---|---|---|
| Ethereum | `ETHEREUM` | Base、Base Sepolia、Ethereum メインネット | Coinbase、Stripe |
| Solana | `SOLANA` | Solana、Solana Devnet | Coinbase、Stripe |

**チェーン(x402 のチャレンジ、`allowed_networks`、残高の照会):**

| チェーン | CAIP-2 の識別子 | 残高 API の値 | 種類 | プロバイダー |
|---|---|---|---|---|
| Base Sepolia | `eip155:84532`(または `base-sepolia`) | `BASE_SEPOLIA` | **テストネット** | Coinbase |
| Base | `eip155:8453` | `BASE` | メインネット | Coinbase |
| Ethereum | `eip155:1` | `ETHEREUM` | メインネット | Coinbase、Stripe |
| Solana | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | `SOLANA` | メインネット | Coinbase、Stripe |
| Solana Devnet | `solana-devnet` | `SOLANA_DEVNET` | **テストネット** | Stripe |

**Base Sepolia**(ファミリー `ETHEREUM`、チェーン `eip155:84532`)から始めてください。<https://faucet.circle.com/> から、無料のテストネットの USDC を入手できます。`init-config` が USDC のコントラクトを知っているのは、Base のチェーンだけです。別のチェーンを追加するには、アドレスを渡すのではなく、そのチェーンの正確なコントラクトを、管理用 CLI の `KNOWN_USDC` に意図的に追加します。

## 運用上のモニタリング

- CloudTrail が `bedrock-agentcore` の呼び出し、特に `ProcessPayment` を記録していることを確認する
- ランタイムのプリンシパルによる `CreatePaymentSession` にアラームを設定する — それは不可能なはずなので、発生した場合は、IAM の分離が崩れたことを意味する
- 繰り返される `ProcessPayment` の失敗に、不正利用の兆候としてアラームを設定する
- 支払いが予期せず拒否され始めたときは、`show-config` の出力を確認する
