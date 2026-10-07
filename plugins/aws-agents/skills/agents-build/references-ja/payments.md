# payments

エージェントに AgentCore Payments を追加します。これは、エージェントがマイクロトランザクション（Coinbase CDP、Stripe Privy）を通じて、x402 と MPP で保護された API、MCP ツール、Web コンテンツの料金を支払えるようにするマネージドサービスです。

AgentCore Payments は**プロトコル非依存**です。**x402**（Coinbase/Cloudflare による、HTTP ネイティブなステーブルコインのマイクロペイメントプロトコル）と、**MPP**（Stripe と Tempo による Machine Payments Protocol）の両方をサポートしています。どちらも同じ `ProcessPayment` API と、同じ manager/connector/instrument/session リソースを通じて処理されます。サービスは、加盟店の `402 Payment Required` レスポンスから、その加盟店がどちらのプロトコルを使っているかを検出し、対応する支払いの証明を生成します。事前にプロトコルを選ぶ必要はありません。Payments を一度プロビジョニングすれば、エージェントはどちらの種類の加盟店にも支払えます。2 つの通信フローについては、以下の **How x402 Payment Works** と **MPP (Machine Payments Protocol)** を参照してください。

コントロールプレーンのリソース（payment manager、connector、認証情報プロバイダー）は、AgentCore の **CLI** でプロビジョニングします。ユーザー単位のデータプレーンのリソース（instrument、session）は、AgentCore の **SDK**（提供されているスクリプト）で作成します。Payments をエージェントに組み込む方法は 2 つあります。(1) Strands（プラグイン）または LangGraph（ミドルウェア）向けの**フレームワークネイティブな統合**。402 の検出、支払いの署名、リトライを透過的に処理するため、独自のツールコードは不要です。(2) その他の Python フレームワーク（OpenAI Agents SDK、CrewAI など）向け、または完全に手動で制御したい場合向けの、**フレームワーク非依存のローカルツール**（`scripts/process_payment_tool.py`）。

## 使うタイミング

- エージェントに、x402 または MPP で保護されたコンテンツ（API、MCP ツール、ペイウォールのあるサイト）の料金を自律的に支払わせたい
- ツール呼び出しが `402 Payment Required` を返し、それを自動的に決済してリトライさせたい
- payment manager があり、エージェントのコードに Payments を組み込む必要がある
- エージェントが使える金額の予算を管理したい
- 支払いの処理が期待どおりに動作しない

次の場合には、このスキルを使わないでください:

- Gateway を通じて、有料ではない外部のツール/API に接続する → `agents-connect` を使います
- インバウンド認証（誰がエージェントを呼び出せるか）→ `agents-harden` を使います
- 一般的なエージェントのスキャフォールディングやプロジェクトの作成
- 支払いとは関係のないエージェントの機能（Memory、VPC、マルチエージェント）

## 入力

`$ARGUMENTS` は省略可能です: `/payments`、`/payments wire`、`/payments debug`、`/payments coinbase`、`/payments stripe`。

## 手順

**実行モデル — 人間が介入する場面を最小限にする。** 各ステップは、間で止まらずに、順番に自分で実行します。開発者が必要になる箇所は**2 か所だけ**です。そこで一時停止し、開発者が確認したら自動的に再開します。

- **ステップ 3b（connector の認証情報）** — 開発者が connector のコマンドを実行します（シークレットが関わるため）。コマンドを提示して、完了の確認を待ちます。
- **ステップ 7（委任と入金）** — 開発者がウォレットを承認して入金します（ブラウザとフォーセット）。手順を提示して、待ちます。

それ以外のすべて — ステップ 0〜3a、**4（デプロイ）、5（組み込み）、6（instrument/session）、8（環境変数の設定とテスト）** — は自動で実行します。開発者が 3b の完了を確認したら、最初のウォレットの**ユーザー ID**と**メールアドレス**を尋ね（ステップ 6 で必要です）、その後は 4 → 5 → 6 と、各ステップで許可を求めることなく続けて進め、ステップ 7 を提示します。7 の完了が確認されたら、ステップ 8 を実行します。ステップごとに止まらないでください。

### ステップ 0: AgentCore CLI をインストール/確認する

CLI は、**npm** パッケージの `@aws/agentcore`（Node.js 20 以上）です。pip パッケージではありません。`pip install` しないでください。

```bash
agentcore --version        # 0.20.0 以上が必要（Payments のコマンドはプレビューで、0.20.x で追加された）
# なければ、または古ければ:
npm install -g @aws/agentcore
```

### ステップ 1: AgentCore プロジェクトを用意する（CLI でのプロビジョニング用）

CLI は、プロジェクト（`agentcore/agentcore.json`）の中に Payments のリソースをプロビジョニングします。

- **プロジェクトがある場合**: `agentcore/agentcore.json` を読み、`payments` 配列と `runtimes` 配列（フレームワーク）を確認します。
- **プロジェクトがない場合**: 新規に作成します（`--help` は呼ばず、直接実行します）。非対話形式の場合:

  ```bash
  agentcore create --project-name <ProjectName> --name <AgentName> --framework Strands --defaults
  ```

  非対話形式では、`--project-name` と `--name` の両方が必須です（`--name` はエージェント/リソースの名前です。これがないと、CLI は対話式のウィザードに切り替わります）。プロジェクト名は、英字で始まり、英数字のみ、23 文字以内で、アンダースコアは使えません。`--defaults` は Python + Bedrock で、Memory なしです。または、`agentcore create` を実行して対話式のウィザードを使います。プロジェクトが必要なのは、CLI で Payments のリソースをプロビジョニングするときだけです。ローカルの支払いツール（ステップ 5）は、どのエージェント、フレームワーク、ホストでも動作します。

### ステップ 2: 状況を判断する

- **ケース A — 何も設定されていない**: ステップ 3 に進みます。
- **ケース B — manager/connector はあり、組み込みが必要**: ステップ 5 に進みます。
- **ケース C — 組み込み済みで、デバッグ中**: 何が失敗しているか質問し、デバッグのセクションを使います。
- **ケース D — プロジェクトなしで Payments について質問されている**（アーキテクチャ、フローの説明）: 支払いのエンドツーエンドのフロー（x402 は **How x402 Payment Works**、MPP は **MPP (Machine Payments Protocol)** を参照）を説明し、Payments をセットアップしたいか（→ ステップ 3 に進む）、組み込みの支援が必要か（→ ステップ 5）を尋ねます。

**フレームワークの確認**: プロジェクトが **Strands** または **LangGraph** を使っている場合（`agentcore/agentcore.json` → `runtimes` 配列を確認）、より簡単なネイティブ統合のパス（ステップ 5a）を提案します。独自のツールスクリプトは不要です。別のフレームワークを使っている場合や、開発者が手動で制御したい場合は、汎用のツールのパス（ステップ 5b）を使います。

### ステップ 3: payment manager と connector をプロビジョニングする（CLI — コントロールプレーン）

**3a. Payment manager — シークレットは不要なので、直接実行する（非対話形式）。** エージェントが、開発者の代わりにこれを実行できます。

```bash
agentcore add payment-manager \
--name <ManagerName> \
--network-preferences eip155:84532
```

`eip155:84532` は Base Sepolia（テストネット）です。名前: 英数字とアンダースコア、48 文字以内、先頭は英字。

その後、プロジェクトに、このスキルでオンボードしたことを示すタグを付けます。`agentcore/agentcore.json` を編集し、最上位の `tags` オブジェクトにエントリを 1 つ追加します（既存のエントリは残します）。

```json
"tags": {
  "agentcore:created-by": "agentcore-cli",
  "agentcore:project-name": "<existing value>",
  "agentcore:onboarding-source": "agent-toolkit-skill"
}
```

プロジェクトのタグは、デプロイ時にプロビジョニングされる AWS リソースに適用されます。`agentcore:onboarding-source` タグによって、AgentCore Payments サービスは、このスキルを通じてオンボードされたリソースと、CLI で直接プロビジョニングされたリソースを区別できます。示したとおりに、正確に設定してください。

**3b. Payment connector — プロバイダーの認証情報が必要。これは開発者が実行するもので、エージェントは実行しない。** エージェントは前提条件と下のコマンドを提示しますが、これを実行したり、認証情報を扱ったりしては**いけません**。この 1 つのコマンドで、認証情報プロバイダーと connector が作成されます。CLI は、プロバイダーのシークレットを **`agentcore/.env.local` に平文で**書き込み、認証情報をローカルに記録します。その後、`agentcore deploy`（ステップ 4）がそれらを **AgentCore Identity** にアップロードします（`agentcore.json` には参照だけが残ります）。プロバイダーのシークレットが使われるのはここだけで、後のステップで再利用されることはありません。

**実行前に — プロバイダーの認証情報を取得する**（最初にこれを行ってください。connector のコマンドで必要になります）:

- **Coinbase CDP**（<https://portal.cdp.coinbase.com/>）:
    1. Coinbase Developer Platform のアカウントとプロジェクトを作成する、またはログインする
    2. API キーを生成する（または既存のものを再利用する）— **API Key ID** と **API Key Secret** を控える
    3. **Wallet Secret**（トランザクションの署名などの暗号ウォレット操作用）を生成する
    4. Project > Wallet > Embedded Wallets > Policies で、**Delegated signing を有効にする**（必須）
- **Stripe Privy**（<https://dashboard.privy.io/>）:
    1. AgentCore 専用の Privy アプリを**新規に**作成する（他の用途に使っているアプリを再利用しない）
    2. アプリの設定から **App ID** と **App Secret** をコピーする
    3. Wallet Infrastructure > Authorization > New Key に移動し、P-256 の鍵ペアを生成する
    4. 秘密鍵には `wallet-auth:` というプレフィックスが付いている — **このプレフィックスを削除し**、生の base64 の内容（`MIGHAgEA...` で始まるもの）だけを使う
    5. 鍵と一緒に表示される **Authorization ID**（署名者 ID）を控える

推奨 — 対話式のウィザード。**フラグを付けずに**コマンドを実行します（シークレットは、コマンド、シェルの履歴、プロセス一覧に現れません。ただし、どちらの場合でも CLI は `agentcore/.env.local` にそれらを書き込みます — 下のセキュリティに関する注記を参照してください）。`--manager`/`--name`/`--provider` を渡しても、ウィザードは起動**しません**。それらのフラグを渡すと CLI は非対話モードに切り替わり、すべてのシークレットのフラグも必須になって、足りなければ "Missing required options" で失敗します。

```bash
agentcore add payment-connector
# ウィザードが、すべてを対話形式で尋ねる — manager、connector 名、プロバイダー、そしてシークレット:
#   CoinbaseCDP : API Key ID、API Key Secret、Wallet Secret
#   StripePrivy : App ID、App Secret、Authorization Private Key、Authorization ID
```

非対話形式の代替手段（CI/スクリプト向け）— シークレットをフラグとして渡します。これらはシェルの履歴とプロセス一覧に残るため、ローカルのセットアップではウィザードを推奨します。

```bash
# Coinbase CDP（ダミーの値 — 自分のものに置き換える）
agentcore add payment-connector --manager <ManagerName> --name <ConnectorName> --provider CoinbaseCDP \
--api-key-id 11111111-2222-3333-4444-555555555555 \
--api-key-secret cdp_sk_EXAMPLEexampleEXAMPLEexampleEXAMPLE0000 \
--wallet-secret  cdp_wallet_EXAMPLEexampleEXAMPLEexample1111
# Stripe Privy（ダミーの値 — 自分のものに置き換える）
agentcore add payment-connector --manager <ManagerName> --name <ConnectorName> --provider StripePrivy \
--app-id clxxxxxxxxxxxxxxxxxxxxxxxx \
--app-secret privy_sk_EXAMPLEexampleEXAMPLEexample2222 \
--authorization-private-key MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHkwEXAMPLE... \
--authorization-id ezzzzzzzzzzzzzzzzzzzzzzzz
```

> **ウィザードとフラグの違い:** フラグ `--manager`、`--name`、`--provider` には `[non-interactive]` と記されています。これらのどれかを指定すると、CLI は**非対話モード**に切り替わり、**必須のシークレットをすべてフラグで**渡すことを求めます。これら 3 つのフラグを付けて、シークレットを省略すると、ウィザードに戻るのではなく、必須フラグが足りないというエラーになります。対話式のウィザードを使うには、フラグなしで `agentcore add payment-connector` を実行してください。その後、開発者が完了を確認するのを待ちます。

セキュリティ:

- **`agentcore/.env.local` には、プロバイダーのシークレットが平文で入っています。** CLI は、connector を追加したとき（ウィザードでもフラグでも）にこのファイルを書き込み、`agentcore deploy` のときに AgentCore Identity にアップロードします。このファイルが gitignore されていることを確認してください。Python のスキャフォールドのデフォルトの `.gitignore` は `.env` しか列挙していないため、`.env.local`（または `.env.*`）を追加してください。エージェントは `agentcore/.env.local` を読んではいけません。
- エージェントはコマンドを提示しますが、実行したり、認証情報を扱ったりすることはありません。認証情報をチャットに貼り付けないでください。

### ステップ 4: デプロイ（リソースの作成）— エージェントが実行する

```bash
agentcore deploy -y
```

`agentcore deploy` は、プロジェクトのリソースを AWS アカウントにプロビジョニングします。payment manager/connector は AgentCore のコントロールプレーン経由で、補助的な IAM（`Payment<Name>ProcessPaymentRole`）とランタイムは、CloudFormation スタック（CDK）経由で作成されます。デプロイ後、manager の ARN、connector ID、ロールの ARN が `agentcore/.cli/deployed-state.json` に書き込まれます。CLI 0.20.x では、これらは `targets.<target>.resources.payments[]` の下（`managerArn`、`connectors[].connectorId`、`processPaymentRoleArn`）にあります。ステップ 6 のスクリプトは、この形式を自動的に読み取ります。

### ステップ 5: エージェントに組み込む

#### ステップ 5a: ネイティブ統合（Strands または LangGraph）— エージェントが実行する

プロジェクトが Strands または LangGraph を使っている場合は、フレームワークのネイティブな Payments 統合を使います。汎用のツールより簡単です。`process_payment_tool.py` も、`x402_fetch` の登録も不要で、ミドルウェア/プラグインが、専用の支払いツールだけでなく、すべてのツール呼び出しを自動的に処理します。

**Strands:**

```python
from strands import Agent
from strands_tools import http_request
from bedrock_agentcore.payments.integrations.config import AgentCorePaymentsPluginConfig
from bedrock_agentcore.payments.integrations.strands.plugin import AgentCorePaymentsPlugin

config = AgentCorePaymentsPluginConfig(
    payment_manager_arn=os.environ["PAYMENT_MANAGER_ARN"],
    user_id=os.environ["PAYMENT_USER_ID"],
    payment_instrument_id=os.environ["PAYMENT_INSTRUMENT_ID"],
    payment_session_id=os.environ["PAYMENT_SESSION_ID"],
    region=os.environ.get("AWS_REGION", "us-west-2"),
)
plugin = AgentCorePaymentsPlugin(config=config)
agent = Agent(
    system_prompt="You are a helpful assistant that can access paid APIs.",
    tools=[http_request],
    plugins=[plugin],
)
```

プラグインは、どのツールからの 402 レスポンスもインターセプトし、支払いに署名して、自動的にリトライします。特別なツールは不要で、エージェントは `http_request` を普通に使うだけです。

**LangGraph:**

```python
from langchain.agents import create_agent
from bedrock_agentcore.payments.integrations.langgraph import (
    AgentCorePaymentsConfig,
    AgentCorePaymentsMiddleware,
)

# 次の設定のうち、どれか 1 つを選ぶ

# オプション A: 明示的なセッション（本番）
  config = AgentCorePaymentsConfig(
      ...
      payment_session_id=os.environ["PAYMENT_SESSION_ID"],
  )
  
  # オプション B: 自動セッション（開発/テスト向けの手軽な方法）
  config = AgentCorePaymentsConfig(
      ...
      auto_session=True,
      auto_session_budget="5.00",
      auto_session_expiry_minutes=60,
  )

payments = AgentCorePaymentsMiddleware(config)

agent = create_agent(
    model=model,
    tools=[],  # ミドルウェアが http_request と支払い照会ツールを自動で登録する
    middleware=[payments],
)
```

ミドルウェアはすべてのツール呼び出しをラップし、どのようなレスポンス形式からでも 402 を検出し（`PAYMENT_REQUIRED:` マーカーは不要）、支払いに署名してリトライします。また、`http_request` ツールと支払い照会ツールも自動で登録します。

**汎用ツールのパスと比べた LangGraph の簡略化:**

- `process_payment_tool.py` スクリプトは不要 — ミドルウェア自体が支払いツールです
- 特別なシステムプロンプトは不要 — 有料の URL に特定のツールを使うようモデルに指示する必要はありません。すべてのツールが支払いに対応しています
- `auto_session=True` により、最初の 402 でセッションを遅延的に作成できます（開発/テスト向けの手軽な方法 — ランタイムロールに `CreatePaymentSession` の IAM 権限が必要）
- エラーからの回復 — 任意の `on_payment_error` コールバックにより、LLM にエラーを見せずに、プログラムで回復できます（新しいセッションの作成、instrument の入れ替え）

> **`auto_session` についての注記**: これは、ミドルウェアのインスタンスごとに、開発者が設定した予算でセッションをちょうど 1 つ作成します。LLM がこれを起動したり制御したりすることはできません。IAM ロールを分離した本番環境（推奨される ProcessPaymentRole）では、`CreatePaymentSession` の呼び出しは拒否されます。代わりに、明示的な `payment_session_id` を使用してください。[IAM roles for AgentCore payments](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-iam-roles.html) を参照してください。

#### ステップ 5b: フレームワーク非依存のローカルツール（任意のフレームワーク）— エージェントが実行する

Payments は、フレームワーク固有のプラグインではなく、小さなローカルツールで組み込みます。そのため、同じコードがどのフレームワークでも動作します。

1. **[`scripts/process_payment_tool.py`](../scripts/process_payment_tool.py) をエージェントのプロジェクトにコピーします。** これは `x402_fetch(url, method="GET")` を公開しています。`402` を受け取ると、SDK の `PaymentManager.generate_payment_header` を呼び出します。SDK が 402 を検証し、ネットワークを選択し、支払いを処理して、バージョンに応じた証明（v1 は `X-PAYMENT`、v2 は `PAYMENT-SIGNATURE`）を構築します。その後、新しいクライアントでリトライします。Base Sepolia での決済は、断続的に一時的な失敗が起きます（ヘッダーは有効なのに、支払い後のリトライでも 402 が返ります）。そのためこのツールは、あきらめる前に、決済とリプレイのフローを最大 `X402_MAX_PAYMENT_ATTEMPTS` 回（デフォルトは 5、環境変数で上書き可能）やり直します。これらのリトライを通じて、冪等性トークンを 1 つだけ再利用するため、`ProcessPayment` は冪等なままです。すべての試行で、同じオンチェーンの承認/nonce が再送されるので、ユーザーに二重に課金されることはありません（リトライは、まだ決済されていない支払いを決済するか、すでに決済済みの場合はオンチェーンで失敗します）。設定は環境変数（ステップ 8 で設定）から読み取ります: `PAYMENT_MANAGER_ARN`、`PAYMENT_INSTRUMENT_ID`、`PAYMENT_SESSION_ID`、`PAYMENT_USER_ID`、`AWS_REGION`。

2. **`x402_fetch` を、エージェントのフレームワークにツールとして登録します。** ツールの関数は同一で、登録のデコレーターだけが異なります。

   ```python
   # Strands
   from strands import Agent, tool
   from process_payment_tool import x402_fetch as _x402
   x402_fetch = tool(_x402)
   agent = Agent(model=..., tools=[x402_fetch], system_prompt="... use x402_fetch for paid URLs ...")
   ```

   ```python
   # LangGraph
   from langchain_core.tools import tool
   from langgraph.prebuilt import create_react_agent
   from process_payment_tool import x402_fetch as _x402
   graph = create_react_agent(model, tools=[tool(_x402)])
   ```

   ```python
   # OpenAI Agents SDK
   from agents import Agent, function_tool
   from process_payment_tool import x402_fetch as _x402
   agent = Agent(name="PaymentAgent", tools=[function_tool(_x402)], instructions="... use x402_fetch for paid URLs ...")
   ```

   その他のフレームワークでは、そのフレームワークのツールの仕組みを使って `x402_fetch` を登録してください。この関数は単なる Python です。

エージェントは、汎用の HTTP ツールではなく `x402_fetch` を呼び出し、支払いはツールの内部で処理されます。（支払いが必要になる可能性のある URL には `x402_fetch` を使うよう、システムプロンプトでモデルに伝えてください。）

### ステップ 6: ユーザー単位の instrument と session をプロビジョニングする（SDK スクリプト — データプレーン）— エージェントが実行する

instrument（ユーザー単位のウォレット）と session（予算で制限された支出期間）は、データプレーンのリソースです。これらには CLI コマンドがありません。まず、ウォレットをプロビジョニングする対象の**ユーザー ID**と**メールアドレス**を開発者に尋ねます（ステップ 3b の後にまだ収集していない場合）。その後、提供されているスクリプト [`scripts/setup_payment_user.py`](../scripts/setup_payment_user.py) を、ユーザーごとに一度実行します。このスクリプトは、`deployed-state.json` から manager の ARN/connector ID を自動で読み取ります（または `--manager-arn`/`--connector-id` を受け付けます）。

```bash
python scripts/setup_payment_user.py --user-id alice --email alice@example.com --budget 5
```

これは instrument（`linkedAccounts` にメールアドレスを設定）と、予算で制限された session を作成し、`PAYMENT_INSTRUMENT_ID` / `PAYMENT_SESSION_ID` / `PAYMENT_USER_ID` の `export` 行（ステップ 8 で使用）と、`wallet_address` と `redirect_url`（ステップ 7 で使用）を表示します。このスクリプトが、データプレーンの正規の手順です。SDK の呼び出しを手で書かないでください。

**`auto_session=True` を使う LangGraph**: ステップ 5a で LangGraph を使い、`auto_session=True` を設定した場合、このステップで必要なのは instrument だけです。session の作成はスキップしてください。ミドルウェアが、最初の 402 で自動的に session を作成します。instrument のために `setup_payment_user.py` を実行する必要はあります（`--budget` フラグは使わないでください。session が作成されてしまいます）。

### ステップ 7: 委任と入金（ウォレットごとに一度だけ）— 開発者が行う

スクリプトが表示した `wallet_address` / `redirect_url` を使います。

1. **委任** — エージェントがウォレットから支払うことを承認します。
    - **Coinbase CDP**: エンドユーザーが `redirect_url` にアクセスし、ログインして、`wallet_address` に権限を付与します。
    - **Stripe Privy**: `redirect_url` はありません。Privy のフロントエンド SDK（<https://github.com/privy-io/aws-agentcore-sdk>）を使い、エンドユーザーのメールアドレスでログインして、委任を承認します。

2. **入金** — Circle のフォーセット（<https://faucet.circle.com/>、Base Sepolia）から、`wallet_address` にテストネットの USDC を送ります。

### ステップ 8: 環境変数を設定してテストする — エージェントが実行する

ステップ 6 のスクリプトが表示した `export` 行から、ツールの設定を行います。スクリプトはすべて（`PAYMENT_MANAGER_ARN`、`PAYMENT_INSTRUMENT_ID`、`PAYMENT_SESSION_ID`、`PAYMENT_USER_ID`、`AWS_REGION`）を出力するので、エージェントの環境にそのままコピーします。

```bash
export PAYMENT_MANAGER_ARN=...      # setup_payment_user.py が表示する 5 つすべて
export PAYMENT_INSTRUMENT_ID=...
export PAYMENT_SESSION_ID=...
export PAYMENT_USER_ID=...
export AWS_REGION=...
```

**LangGraph（`auto_session=True` を使うステップ 5a）**: 必要な環境変数は次のとおりです。

```bash
export PAYMENT_MANAGER_ARN=...
export PAYMENT_INSTRUMENT_ID=...
export PAYMENT_USER_ID=...
export AWS_REGION=...
# PAYMENT_SESSION_ID は不要 — auto_session が内部で管理する
```

エージェントを実行し、有料のエンドポイントを取得するよう指示します。

```
Fetch https://sandbox.node4all.com/v1/x402-test and tell me what you find.
```

エージェントの実行方法は問いません。フレームワークで直接実行するか、ローカルサーバーなら `agentcore dev`、デプロイ済みのランタイムなら `agentcore invoke` を使います（ランタイムにも同じ `PAYMENT_*` 環境変数を設定してください）。成功すると、`x402_fetch` が `402` を受け取り、支払いを決済し、リトライが `200` を返す様子が確認できます。

## `upto` スキームと Permit2 のアローワンス

上記のセットアップと組み込みは、スキームに依存しません。エージェントは、加盟店の `402` が提示する x402 のスキームをそのまま通します。ほとんどのエンドポイントは `exact`（事前に分かっている固定価格）を使います。一部は **`upto`** を使い、これは従量課金や使用量ベースの価格設定（推論ごとの支払いなど）向けです。エージェントは支出の上限を承認し、加盟店は、その上限までの範囲で実際に消費した金額を決済します。`upto` のエンドポイントに支払うために、設定を変更する必要はありません。

`upto` スキームには、追加の前提条件が 1 つあります。[Uniswap Permit2](https://docs.uniswap.org/contracts/permit2/overview) コントラクトを通じて決済するため、支払い元のウォレットがそのアセットの Permit2 アローワンスを持っている必要があります。任意の **`permit2_allowance_limit`** フィールドは、このアローワンスを付与する `upto` 向けのアドオンです。これを設定すると、`ProcessPayment` は署名の前に、一度だけのオンチェーンの `approve` を送信します。ネイティブ統合の設定（ステップ 5a）で設定します。

```python
config = AgentCorePaymentsPluginConfig(   # LangGraph の場合は AgentCorePaymentsConfig
    ...,
    permit2_allowance_limit="1000000",     # 10 進数の文字列、アセットの最小単位（1000000 = 1 USDC）
)
```

- アセットの最小単位で表した、10 進数の文字列です。uint256 の最大値（`115792089237316195423570985008687907853269984665640564039457584007913129639935`）は、無制限のアローワンスを付与します。
- 適用されるのは `upto` だけです。`exact` の支払いに指定すると、検証エラーが返されます。
- 実際のオンチェーンの `approve` トランザクションをブロードキャストします。ガス代は、ウォレットのネイティブトークンの残高から支払われます（USDC の残高ではありません）。
- 必要なのは、ウォレットで最初に `upto` の支払いを行うときだけです。`approve` は、アローワンスを加算するのではなく設定するため、以降の呼び出しでは、無駄なトランザクションを避けるために省略してください。
- 統合の設定（または `generate_payment_header`）が `permit2_allowance_limit` を受け付ける SDK のビルドが必要です。

汎用の `x402_fetch` ツール（ステップ 5b）では、`upto` のエンドポイントに支払うとき、`generate_payment_header` の呼び出しに `permit2_allowance_limit="..."` を渡します。

## Payments のデバッグ

**エージェントが 402 を受け取っても支払わない:**

1. 環境変数 `PAYMENT_MANAGER_ARN` が設定されていて、None でないことを確認する
2. エージェントが、汎用の `http_request` ではなく `x402_fetch` ツールを使っていることを確認する
3. x402 のチャレンジが、レスポンスボディ（`x402Version` と `accepts` のフィールド）または `payment-required` ヘッダーのどちらかに存在することを確認する

**ProcessPayment が "Failed to obtain resource payment token" で失敗する:**

- IAM サービスロールに権限が不足しています。トークンボールトに対する `GetResourcePaymentToken` と、シークレットに対する `secretsmanager:GetSecretValue` があることを確認してください。
- ロールを作成してから、ProcessPayment を呼び出すまでに 15 秒以上待ってください（IAM の反映待ち）。

**ProcessPayment が "Failed to obtain workload access token" で失敗する:**

- サービスロールに、workload-identity-directory のリソースに対する `GetWorkloadAccessToken` 権限がありません。

**ProcessPayment が "Failed to assume payment execution role" で失敗する:**

- サービスロールの信頼ポリシーが正しくありません。正しい `aws:SourceAccount` 条件を付けて、`bedrock-agentcore.amazonaws.com` を信頼していることを確認してください。
- Payment Manager に渡したロールの ARN が、実際のロールと一致していることを確認してください。

**ProcessPayment は成功するが、加盟店が依然として 402 を返す:**

- **一時的なオンチェーン決済の失敗**（Base Sepolia でよくある）: ツールはすでに、最大 `X402_MAX_PAYMENT_ATTEMPTS` 回（デフォルトは 5）、再決済しています。それでも 402 が返る場合は、上限を引き上げる（`export X402_MAX_PAYMENT_ATTEMPTS=8`）か、少し時間をおいてリトライしてください。
- **Cookie の混入**: リトライが、最初の 402 のリクエストで受け取った Cookie を送信しています。新しい httpx クライアントを使ってください: `httpx.Client(cookies=None).request(...)`。同じクライアント/セッションを再利用**しないでください**。
- **x402 のバージョン/ヘッダーの誤り**: 加盟店は x402 v2 なのに、証明が v1 として送られた（またはその逆）。v1 は、フラットな証明（最上位の `scheme`/`network`）を持つ `X-PAYMENT` ヘッダーを期待します。v2 は、`accepted` が `payload` と同じ階層の最上位にあり、`payload` には `signature` と `authorization` のみが含まれる（最上位の `scheme`/`network` はない）`PAYMENT-SIGNATURE` ヘッダーを期待します。v2 の加盟店が v1 の `X-PAYMENT` ヘッダーを受け取ると、それを無視して同じ 402 を再発行します。多くの場合、ボディは空の `{}` で、エラーは出ないため、診断が難しくなります。チャレンジ（ボディまたは `payment-required` ヘッダー）から `x402Version` を読み取り、対応する証明を構築してください。
- **証明の形式の不一致（network フィールド）**: **v1** では、証明の `network` に、加盟店の人間が読めるラベルを使う必要があります（例: `"eip155:84532"` ではなく `"base-sepolia"`）。**v2** では、証明はチャレンジにある CAIP-2 識別子をそのまま保持します（例: `"eip155:84532"`）。注意: `ProcessPayment` への入力は、バージョンに関係なく常に CAIP-2 を使います。異なるのは、加盟店に提示する証明だけです。
- **証明の期限切れ**: 証明には、約 60 秒の有効期間（`validBefore`）があります。エージェントのループが遅い場合、リトライの前に証明が期限切れになることがあります。

**ProcessPayment は成功する（PROOF_GENERATED）が、加盟店が空の `{}` ボディでエラーなしの 402 を返す:**

- 加盟店は x402 の **v2** で、v1 の `X-PAYMENT` ヘッダーを無視しています。チャレンジ（ボディまたは `payment-required` レスポンスヘッダーにある `x402Version: 2`）からバージョンを検出し、`PAYMENT-SIGNATURE` ヘッダーを送ってください。v2 の証明では、`accepted`（要件全体。network は CAIP-2）が `payload` と同じ階層の最上位に置かれ、`payload` には `signature` と `authorization` のみが含まれます。注意: ProcessPayment が `PROOF_GENERATED` を返し、証明の形式も正しいのに加盟店が 402 を返す場合は、一時的なオンチェーン決済の失敗かもしれません。形式の問題だと決めつける前に、一度リトライしてください。

**MPP: `ProcessPayment` が、ガス/ネットワーク手数料に言及した `ValidationException` で失敗する:**

- MPP のチャレンジが、売り手が負担する手数料を提示していません（`methodDetails.feePayer` が `false` か、存在しない）。そのため AgentCore は、買い手にネットワーク手数料を黙って課すことはしません。`paymentInput.mpp.buyerPaysGasFees: true` を設定して、買い手のウォレットから支払うことを承認するか、売り手が手数料を負担するチャレンジを取得してください。

**MPP: `ProcessPayment` が、チャレンジのヘッダーに関する `ValidationException` で失敗する:**

- `wwwAuthenticateHeaders` には、生の `WWW-Authenticate: Payment …` の値を**そのまま**、**ちょうど 1 つ**だけ含める必要があります。デコード、再構成、再エンコードをしないでください。バイト列を変更すると、チャレンジの HMAC バインディングが壊れます。`402` が複数の `WWW-Authenticate: Payment` 行を返した場合は、instrument が満たせる 1 つのオプションだけを送ってください。
- `paymentType` が `MPP` で、ペイロードが `paymentInput` の `mpp` アーム（`cryptoX402` ではない）の下にあることを確認してください。

**MPP: 認証情報（credential）を付けてリトライしたのに、エージェントが新しい `402` を受け取る:**

- 認証情報は、返されたとおりに付けてください: `Authorization: Payment <token>`。`paymentOutput.mpp.paymentCredential` を、そのまま使います（すでに `Payment` スキームのプレフィックスが含まれています）。最初の `402` の Cookie が再送されないように、新しい HTTP クライアントでリトライしてください。
- MPP の認証情報は一度しか使えません。すでに消費された `challenge.id` を再送すると、拒否されます。新しい `402` から得た新しいチャレンジに対して、`ProcessPayment` をやり直してください。

**MPP: `ProcessPayment` が `SubscriptionRequiredException`（403）で失敗する:**

- アカウントが、必要な AWS Marketplace のオファーをサブスクライブしていません。エラーに含まれる `subscriptionUrl` に従ってサブスクライブしてから、リトライしてください。

**ProcessPayment が "Payment session not found" で失敗する:**

- セッション ID が無効か、セッションが削除されています。新しいセッションを作成してください。
- セッション作成時の `paymentManagerArn` が、ProcessPayment で使うものと一致していることを確認してください。

**ProcessPayment が "PaymentSessionExpired" で失敗する:**

- 支払いセッションには有効期限があります。`expiryTimeInMinutes` を指定して、新しいセッションを作成してください。

**ProcessPayment が "Payment instrument not found" または "does not belong to user" で失敗する:**

- instrument ID が正しく、同じ Payment Manager に属していることを確認してください。
- ProcessPayment に渡した `userId` が、instrument の作成時に使った `userId` と一致していることを確認してください。

**ProcessPayment が "Payment connector is not active" で失敗する:**

- connector がまだプロビジョニング中の可能性があります。状態を確認して、待ってください。
- connector が削除または無効化されている場合は、新しく作成してください。

**ProcessPayment が "Network mismatch" で失敗する:**

- x402 のチャレンジが指定するネットワークが、instrument のネットワークと一致していません。
- `network: "ETHEREUM"` で作成した instrument は、Base、Base Sepolia、Ethereum のチェーンをサポートします。
- `network: "SOLANA"` で作成した instrument は、Solana と Solana Devnet のチェーンをサポートします。

**ProcessPayment が "Payment asset not supported USDC token address" で失敗する:**

- x402 のチャレンジにある USDC のコントラクトアドレスが、そのネットワークで期待されるアドレスと一致していません。
- Base Sepolia の USDC: `0x036CbD53842c5426634e7929541eC2318f3dCF7e`
- サポートされているのは USDC のみです。

**ProcessPayment が "Wallet does not have a USDC balance" で失敗する:**

- ウォレットに、指定されたチェーンの USDC がありません。
- Circle のフォーセット（テストネット）から入金してください: https://faucet.circle.com/
- メインネットの場合: エンドユーザーが、ウォレットに直接入金する必要があります。

**Coinbase: "Delegated signing grant is not active":**

- エンドユーザーが、委任のステップを完了していません。
- instrument の作成時に返された `redirectUrl`（Coinbase Hub）に、リダイレクトしてください。
- ユーザーは、ログインして、ウォレットに権限を付与する必要があります。

**Coinbase: "Delegated signing is not enabled":**

- Coinbase CDP のプロジェクトで、Delegated signing が有効になっていません。
- portal.cdp.coinbase.com > Project > Wallet > Embedded Wallets > Policies > Enable Delegated signing に進んでください。

**Stripe Privy: "Privy credentials are invalid":**

- 認証情報プロバイダーに保存されている App ID または App Secret が間違っています。
- Privy ダッシュボードで、認証情報が一致していることを確認してください。
- 正しい値で、認証情報プロバイダーを作成し直してください。

**Stripe Privy: "Privy appId is invalid or missing":**

- 認証情報プロバイダーの設定にある `appId` が正しくありません。
- Privy ダッシュボードで、正しい App ID を確認してください。

**Stripe Privy: "Privy signing key is invalid or expired":**

- Authorization Private Key または Authorization ID が無効か、期限切れです。
- Privy ダッシュボード > Wallet Infrastructure > Authorization で、新しい P-256 の鍵ペアを生成してください。
- 秘密鍵から `wallet-auth:` プレフィックスを削除するのを忘れないでください。
- 新しい鍵で、認証情報プロバイダーを更新してください。

**Stripe Privy: "Wallet policy denied the transaction":**

- Privy で設定されたウォレットポリシーが、トランザクションをブロックしています。
- Privy ダッシュボードで、ウォレットポリシーの設定を確認してください。
- トランザクションの金額、受取人、頻度が、ポリシーの上限を超えていないか確認してください。

**Stripe Privy: "The linked account data is invalid":**

- instrument の作成時に `linkedAccounts` で使ったメールアドレスまたは電話番号の形式が不正です。
- メールアドレスの形式が正しいことを確認してください。

**Stripe Privy: "Rate limited by Privy":**

- Privy の API が、リクエストをレート制限しています。
- 間隔をあけてリトライしてください。Privy のレート制限に関するドキュメントを確認してください。

**ProcessPayment が "Payment amount exceeds maximum" で失敗する:**

- x402 のチャレンジが、1 回のトランザクションで許可される最大額を超える金額を要求しています。
- チャレンジの金額を確認し、セッションの予算で許容されるか検証してください。

**ProcessPayment が "Rate exceeded" で失敗する:**

- API の呼び出しが多すぎます。数秒おいてからリトライしてください。

**Coinbase: "Delegation not completed":**

- エンドユーザーが、エージェントにウォレットからの支払いの権限を付与していません。
- instrument の作成時に返された `redirectUrl` にアクセスし、ログインして、権限を付与してください。

**Stripe Privy: "Delegation not completed":**

- エージェントの認証キーが、埋め込みウォレットの署名者として追加されていません。
- Privy のフロントエンド SDK（https://github.com/privy-io/aws-agentcore-sdk）を使ったフロントエンドをセットアップし、セットアップ時に指定されたエンドユーザーのメールアドレスでログインして、ウォレットの委任を承認してください。

## セキュリティに関する考慮事項

- **認証情報のローテーション**: 支払いプロバイダーの認証情報を定期的にローテーションしてください。更新した値で、認証情報プロバイダーを作成し直します。
- **予算/支出の上限**: 支払いの暴走を防ぐために、Payment Session の `expiryTimeInMinutes` と、セッションごとの予算制御を使ってください。
- **監査ログ**: CloudTrail が、`bedrock-agentcore` のすべての API 呼び出し、特に `ProcessPayment` を記録していることを確認してください。本番環境では、不正利用の可能性を示す指標として、失敗した支払いの試行に対する CloudWatch アラームを設定してください。
- **SSRF 対策**: `x402_fetch` ツールは、HTTPS のみを強制し、内部エンドポイントの取得を防ぐためにプライベート IP の範囲をブロックします。
- **最小権限**: IAM サービスロールには、必要最小限の権限（トークンボールト、ワークロードアイデンティティ、シークレットへのアクセス）だけを付与してください。
- **セッションの有効期限**: 支払いセッションは短命に保ってください（60 分以内）。長期間有効なセッションを再利用するのではなく、ユーザーとのやり取りごとに新しいセッションを作成してください。
- **転送中の暗号化**: すべての支払いリクエストは HTTPS を使う必要があります。`x402_fetch` ツールは、HTTPS 以外の URL を拒否します。

包括的なセキュリティのガイダンスについては、[AgentCore Security documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/security.html) を参照してください。

## How x402 Payment Works（エンドツーエンド）

```
エージェントが x402_fetch("https://paid-api.example.com/data") を呼び出す
  │
  ├─ 1. HTTP GET → 402 Payment Required
  │     Body: {"x402Version": 1, "accepts": [{"scheme": "exact", "network": "base-sepolia", ...}]}
  │
  ├─ 2. x402 のチャレンジを抽出する
  │
  ├─ 3. ProcessPayment(paymentManagerArn, instrumentId, sessionId, challenge)
  │     → 署名済みの証明（signature + authorization）を返す
  │
  ├─ 4. 支払いヘッダーを構築する（v1 は X-PAYMENT、v2 は PAYMENT-SIGNATURE）
  │
  ├─ 5. 支払いヘッダーを付けてリトライする（新しい HTTP クライアント、Cookie なし）
  │     → 200 OK + 有料のコンテンツ
  │
  └─ 6. コンテンツをエージェントに返す
```

## MPP (Machine Payments Protocol)

MPP は、x402 と並んで AgentCore Payments が扱う、2 つ目の支払いプロトコルです。マシン間の支払いのための、プロトコルに中立で HTTP ネイティブな方式です（IETF のドラフト。<https://mpp.dev> を参照）。AgentCore は**買い手**の側で動作します。エージェントが有料のエンドポイントにアクセスし、`402 Payment Required` レスポンスで MPP のチャレンジを受け取り、`ProcessPayment` を呼び出して、それを満たす認証情報（credential）を生成します。ライフサイクルは x402 と同じですが、通信フォーマットが異なります。

### 通信における MPP と x402 の違い

x402 は、チャレンジをレスポンスの**ボディ**（`x402Version` + `accepts`）で運び、証明を `X-PAYMENT`（v1）または `PAYMENT-SIGNATURE`（v2）ヘッダーで運びます。MPP は、代わりに標準的な HTTP 認証のハンドシェイクを使います。

| プリミティブ | 方向 | HTTP ヘッダー | エンコーディング |
|---|---|---|---|
| **チャレンジ** | サーバー → エージェント（402） | `WWW-Authenticate: Payment ...` | RFC 9110 の auth-params（`id="…", realm="…", method="…", intent="…", request="<base64url>", …`） |
| **認証情報（credential）** | エージェント → サーバー（リトライ） | `Authorization: Payment <token>` | `base64url(JSON)`、パディングなし |
| **レシート** | サーバー → エージェント（200） | `Payment-Receipt: <token>` | `base64url(JSON)`、パディングなし |

各 `402` は、`WWW-Authenticate: Payment` ヘッダーの行を**1 つ以上**含むことがあります。支払いオプションごとに 1 行です（それぞれ異なる `method`/`intent`）。エージェントは、満たせるものを 1 つ選び、ちょうど 1 つの `Authorization: Payment` ヘッダーを返します。`method`（`tempo`、`evm`、`solana`、`stripe`、`card` など）と `intent`（`charge`、`session`、`subscription`）は、オープンな IANA レジストリです。MPP は、メソッドと通貨に依存せず（暗号資産でも法定通貨でも可）、USDC のみの x402 とは異なります。メソッドごとの `request` ペイロードは、不透明な base64url の塊としてチャレンジの中に入っています。AgentCore がそれを解析して対応する証明を生成するので、自分でデコードせずに、チャレンジをそのまま転送します。

### MPP の ProcessPayment の契約

x402 で使うのと同じ `ProcessPayment` オペレーションを呼び出します。`paymentType` に `MPP` を設定し、`paymentInput` の `mpp` アームを使います。

```jsonc
// ProcessPayment のリクエスト（MPP）
{
  "paymentManagerArn": "arn:aws:bedrock-agentcore:us-west-2:111122223333:payment-manager/pm-abc123",
  "paymentSessionId": "payment-session-…",
  "paymentInstrumentId": "payment-instrument-…",
  "paymentType": "MPP",
  "paymentInput": {
    "mpp": {
      "version": "1",
      // 402 から得た、生の WWW-Authenticate: Payment ヘッダーの値。そのまま渡す。
      // このリリースでは、ちょうど 1 つのエントリ（ACP は、1 回の呼び出しで 1 つのチャレンジを満たす）。
      "wwwAuthenticateHeaders": [
        "Payment id=\"qB3…\", realm=\"api.example.com\", method=\"evm\", intent=\"charge\", request=\"eyJ…\""
      ],
      // 任意。買い手がブロックチェーン（ガス）手数料を支払う場合に、ACP が署名することを承認する。
      "buyerPaysGasFees": false
    }
  }
}
```

```jsonc
// ProcessPayment のレスポンス（MPP） — ステータスは PROOF_GENERATED
{
  "paymentType": "MPP",
  "status": "PROOF_GENERATED",
  "paymentOutput": {
    "mpp": {
      "version": "1",
      // 支払ったチャレンジの id をそのまま返す。デコードせずに対応付けができる。
      "selectedPaymentId": "qB3…",
      // そのまま送れる Authorization ヘッダーの値: "Payment <base64url-token>"。
      // そのまま付けて元のリクエストをリトライする — 組み立ては不要。
      "paymentCredential": "Payment eyJ…"
    }
  }
}
```

API モデルに基づく補足:

- **ヘッダーはそのまま転送します。** 生の `WWW-Authenticate: Payment …` の値を、変更せずに渡します。AgentCore が auth-params を自分で解析するので、フィールドの対応付けや base64 の処理は不要です。また、そのまま転送することで、チャレンジの HMAC が結び付けているバイト列が保たれます。
- **1 回の呼び出しにつき、チャレンジは 1 つです。** このリリースの `wwwAuthenticateHeaders` は、ちょうど 1 つのエントリを受け付けます。`402` が複数のオプションを提示した場合は、instrument が満たせるものを選び、その行だけを送ります。（将来、互換性を壊さずに複数のオプションへ契約を広げられるように、リストとしてモデル化されています。）
- **`paymentCredential` は完成した `Authorization` ヘッダーです。** 組み立ては不要で、リトライに `Authorization: Payment <token>` として付けるだけです。これは bearer に似たシークレットなので、ログに出力しないでください。
- **`buyerPaysGasFees` は手数料の負担を制御します。** 暗号資産のチャレンジは、`methodDetails.feePayer` で、ネットワーク（ガス）手数料を誰が支払うかを示します。`true` は売り手が負担、`false` または存在しない場合は、買い手が、金額に上乗せして自分のウォレットから支払います。この追加コストはチャレンジの `amount` に含まれていないため、AgentCore は、買い手がそれを受け入れると仮定しません。チャレンジが売り手負担の手数料を提示していない場合、`buyerPaysGasFees: true` を設定したときにだけ署名し、そうでなければ `ValidationException` で失敗します。手数料を売り手が負担するチャレンジでは、省略するか `false` にしてください。その場合は影響がありません。
- **`version`** は MPP のプロトコルバージョン（`"1"` のような、数字だけの文字列）で、x402 のバージョンとは別のものです。

### MPP のエンドツーエンドのフロー

```
エージェントが https://paid-api.example.com/data に GET する
  │
  ├─ 1. 402 Payment Required
  │     WWW-Authenticate: Payment id="qB3…", realm="api.example.com", method="evm", intent="charge", request="eyJ…"
  │
  ├─ 2. ProcessPayment(paymentType="MPP", paymentInput.mpp.wwwAuthenticateHeaders=[<そのヘッダーをそのまま>])
  │     → status は PROOF_GENERATED、paymentOutput.mpp.paymentCredential = "Payment eyJ…"
  │
  ├─ 3. Authorization: Payment eyJ… を付けてリトライする   （新しい HTTP クライアント、Cookie なし）
  │     → 200 OK + 有料のコンテンツ  （任意で  Payment-Receipt: <token>）
  │
  └─ 4. コンテンツをエージェントに返す
```

## サポートされているネットワーク

2 つの概念があります。**network**（ブロックチェーンのファミリー。instrument の作成時に使用）と、**chain**（個別のチェーン。x402 のチャレンジと残高の照会で使用）です。

**ネットワーク（instrument の作成用）:**

| ネットワーク | instrument の値 | プロバイダー |
|---|---|---|
| Ethereum（Base、Base Sepolia を含む） | `ETHEREUM` | Coinbase、Stripe |
| Solana（Solana Devnet を含む） | `SOLANA` | Coinbase、Stripe |

**チェーン（x402 のチャレンジと残高の照会用）:**

| チェーン | 識別子（x402） | 残高 API の値 | 種類 | プロバイダー |
|---|---|---|---|---|
| Base Sepolia | `base-sepolia` または `eip155:84532` | `BASE_SEPOLIA` | テストネット | Coinbase |
| Base | `eip155:8453` | `BASE` | メインネット | Coinbase |
| Ethereum Mainnet | `eip155:1` | `ETHEREUM` | メインネット | Coinbase、Stripe |
| Solana Mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | `SOLANA` | メインネット | Coinbase、Stripe |
| Solana Devnet | `solana-devnet` | `SOLANA_DEVNET` | テストネット | Stripe |

テストでは、**Base Sepolia**（network: `ETHEREUM`、chain: `BASE_SEPOLIA`）から始めてください。https://faucet.circle.com/ から、無料のテストネットトークンを入手できます。

## 品質基準

- CLI は pip ではなく、`npm install -g @aws/agentcore` でインストールされている
- コントロールプレーン（認証情報プロバイダー、manager、connector）は CLI でプロビジョニングされる。manager は非対話形式で作成し、開発者が関わるのは connector の認証情報の入力だけ
- データプレーン（instrument、session）は、手書きのコードではなく、SDK のスクリプトで作成されている
- プロジェクトが Strands または LangGraph の場合は、より簡単なパスとして、ネイティブ統合（ステップ 5a）を最初に提案している
- 汎用のツールのパス（ステップ 5b）を使うのは、他のフレームワークの場合、または開発者が明示的に手動での制御を望む場合のみ
- Payments は、フレームワークネイティブの統合（ステップ 5a）か、フレームワーク非依存の `x402_fetch` ツール（ステップ 5b）で組み込まれている
- x402 と MPP の両方の加盟店に、同じ manager/connector/instrument/session と、同じ `ProcessPayment` API で支払える — 事前にプロトコルを選ぶことはない
- MPP では、生の `WWW-Authenticate: Payment` ヘッダーを（1 回の呼び出しにつき 1 つ）そのまま転送し、返された `paymentCredential` を `Authorization: Payment <token>` として変更せずに付ける
- 認証情報が、エージェントやチャットを通ることはない
