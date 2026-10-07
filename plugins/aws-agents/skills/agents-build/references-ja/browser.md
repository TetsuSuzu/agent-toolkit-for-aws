# browser

AgentCore Browser ツールを追加して、エージェントが Web ページの操作、フォームへの入力、情報の抽出を行えるようにします。

## 使うタイミング

- エージェントが、API を持たない Web サイトとやり取りする必要がある
- エージェントがフォームへの入力、データのスクレイピング、Web アプリの操作を行う必要がある
- エージェント専用の、セッション単位で分離されたブラウザが欲しい（共有ブラウザではない）
- デバッグや監査のために、ブラウザの動作をライブビュー・録画・リプレイしたい

この資料を使わないケース:

- API を呼び出す場合 — Gateway（`agents-connect`）を使用します
- サンドボックスでコードを実行する場合 — [`code-interpreter.md`](code-interpreter.md) を参照してください
- ユーザーにブラウザベースの UI を提供する場合 — これは別の問題です（Browser ツールではなく AGUI プロトコルの領域です）

## 考え方

Browser ツールは、セッションごとに 1 つ、分離された microVM 内で動作する**マネージド Chrome インスタンス**です。エージェントは WebSocket（CDP — Chrome DevTools Protocol）経由で接続し、自動化フレームワークで操作します。フレームワークは次から選びます。

| フレームワーク | 使うタイミング |
|---|---|
| **Strands `AgentCoreBrowser`** | Strands エージェント内でエージェント主導のブラウジングを行う場合。最も高レベルで、ツール利用にネイティブ対応しています。 |
| **Nova Act** | 各ステップで次の操作を LLM に判断させたい場合（「検索ボックスをクリックし、X を入力して、Enter を押す」）。自由度の高いタスクに最適です。 |
| **Playwright** | 決まったスクリプトによる確定的な自動化。ログインフローや、構造が既知のページのスクレイピングなど、手順が正確に分かっている場合に最適です。 |

Strands エージェントにブラウジングを追加する場合は、`AgentCoreBrowser` を使い、フレームワークの選択は省略してください。内部で Nova Act をラップしており、エージェントツールという考え方にも合います。

Strands を使わない場合は、タスクが自由度の高いものか明確に定義されたものかに応じて、Nova Act（推論主導）と Playwright（スクリプト主導）から選びます。

セッションは**デフォルトでは一時的**です（使用のたびにリセットされます）。デフォルトのタイムアウトは 15 分、最大は 8 時間です。複数のセッションを同時に実行できます。

## 前提条件

- Python 3.10 以上
- `bedrock-agentcore` SDK がインストール済みであること
- `bedrock-agentcore:*Browser*` アクションに対する IAM 権限（本番環境では、対象を自分の browser リソース ARN に絞ります）
- Browser をサポートする AWS リージョン — 最新の一覧はドキュメントで確認してください
- Strands を使う場合: 選択したモデルへのアクセス権（Claude Sonnet 4.x が一般的なデフォルトです）
- Nova Act を使う場合: [nova.amazon.com/act](https://nova.amazon.com/act) で取得した Nova Act API キー（執筆時点では、米国の amazon.com アカウントのみ）

IAM ポリシーの雛形（呼び出し元の ID — ユーザー、ロール、または AgentCore Runtime の実行ロール — にアタッチします）:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "BrowserAccess",
    "Effect": "Allow",
    "Action": [
      "bedrock-agentcore:CreateBrowser",
      "bedrock-agentcore:GetBrowser",
      "bedrock-agentcore:ListBrowsers",
      "bedrock-agentcore:StartBrowserSession",
      "bedrock-agentcore:StopBrowserSession",
      "bedrock-agentcore:GetBrowserSession",
      "bedrock-agentcore:ListBrowserSessions",
      "bedrock-agentcore:ConnectBrowserAutomationStream",
      "bedrock-agentcore:ConnectBrowserLiveViewStream"
    ],
    "Resource": "arn:aws:bedrock-agentcore:<REGION>:<ACCOUNT_ID>:browser/*"
  }]
}
```

IAM アクション名は変更される可能性があるため、最新のものをドキュメントで確認してください。

## パス A — Browser ツールを使う Strands エージェント（ほとんどの場合に推奨）

```python
from strands import Agent
from strands_tools.browser import AgentCoreBrowser

browser_tool = AgentCoreBrowser(region="<REGION>")

agent = Agent(tools=[browser_tool.browser])

result = agent("Find the release date of the latest AgentCore SDK on GitHub.")
print(result.message["content"][0]["text"])
```

インストール: `pip install bedrock-agentcore strands-agents strands-agents-tools`

エージェントがブラウザを使うタイミングを判断し、必要に応じてセッションを開き、終了時に片付けます。内部では `AgentCoreBrowser` が AWS マネージドの `aws.browser.v1` リソースを使用するため、リソースを作成する必要はありません。

**AgentCore Runtime のエントリーポイントに組み込む場合:**

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from strands import Agent
from strands_tools.browser import AgentCoreBrowser
from model.load import load_model  # `agentcore create` が生成します
import os

app = BedrockAgentCoreApp()
REGION = os.getenv("AWS_REGION", "us-west-2")

@app.entrypoint
def invoke(payload, context):
    browser_tool = AgentCoreBrowser(region=REGION)
    agent = Agent(model=load_model(), tools=[browser_tool.browser])
    result = agent(payload.get("prompt", ""))
    return {"response": str(result)}

if __name__ == "__main__":
    app.run()
```

## パス B — 推論主導のタスクに Nova Act を使う

各クリックや入力のステップを LLM に判断させる必要があるタスクに使います。

```python
from bedrock_agentcore.tools.browser_client import browser_session
from nova_act import NovaAct

def run_browser_task(prompt: str, starting_page: str, nova_act_key: str, region: str = "us-west-2"):
    with browser_session(region) as client:
        ws_url, headers = client.generate_ws_headers()
        with NovaAct(
            cdp_endpoint_url=ws_url,
            cdp_headers=headers,
            nova_act_api_key=nova_act_key,
            starting_page=starting_page,
        ) as nova:
            return nova.act(prompt)
```

インストール: `pip install bedrock-agentcore nova-act boto3`

`browser_session` コンテキストマネージャーが開始と停止を処理します。セッションをリークさせないでください。必ずコンテキストマネージャーを使うか、素の `BrowserClient` 呼び出しを try/finally で囲んでください。

**認証情報の扱い:** Nova Act の API キーはシークレットです。AgentCore Runtime 上のエージェントで実行する場合は、認証情報として登録し（`agentcore add credential --name NovaAct --api-key ...`）、`@requires_api_key(provider_name="NovaAct")` で取得します。ランタイムの環境変数には入れないでください。`agents-connect` のパス D を参照してください。

## パス C — スクリプトによる自動化に Playwright を使う

手順が固定されていて、確定的な動作が欲しい場合（ログイン、スクレイピング、自動テストなど）に使います。

```python
import asyncio
from bedrock_agentcore.tools.browser_client import browser_session
from playwright.async_api import async_playwright

async def scrape_title(url: str, region: str = "us-west-2") -> str:
    async with async_playwright() as pw:
        with browser_session(region) as client:
            ws_url, headers = client.generate_ws_headers()
            browser = await pw.chromium.connect_over_cdp(ws_url, headers=headers)
            context = browser.contexts[0]
            page = context.pages[0]
            try:
                await page.goto(url)
                return await page.title()
            finally:
                await page.close()
                await browser.close()

print(asyncio.run(scrape_title("https://example.com")))
```

インストール: `pip install bedrock-agentcore playwright`

同期版（`sync_playwright`）もサポートされています。エージェントのコードが非同期かどうかに応じて選んでください。

## オブザーバビリティ

Browser はデフォルトで可観測です。

- **ライブビュー** — 実行中のセッションを AWS コンソールからリアルタイムで確認できます（Built-in tools → Browser → 対象のセッション → "View live session"）。自動化の操作を、対話的に引き継ぐこともできます。
- **CloudWatch Logs** — `/aws/bedrock-agentcore/browser/*`
- **メトリクス** — `AWS/BedrockAgentCore` 名前空間

**セッション録画**（DOM、クリック、コンソールログ、ネットワーク）は、ブラウザごとのオプトインです。有効にするには:

1. 録画を設定した**カスタムブラウザ**（`aws.browser.v1` ではないもの）を作成します
2. その実行ロールに、録画用バケットへの `s3:PutObject` を付与します
3. 録画は自分の S3 バケットに保存され、AWS コンソールでリプレイできます

マネージドの `aws.browser.v1` リソースは録画**しません**。監査証跡が必要な場合は、カスタムブラウザを使用してください。

## セッションのライフサイクル — 必ず閉じる

```python
# 正しい — コンテキストマネージャー
with browser_session(region) as client:
    ws_url, headers = client.generate_ws_headers()
    ...

# これも正しい — 明示的な try/finally
client = BrowserClient(region=region)
client.start()
try:
    ...
finally:
    client.stop()

# 誤り — セッションがリークする
client = BrowserClient(region=region)
client.start()
...  # ここで例外が発生すると、セッションは 15 分のタイムアウトまでアイドル状態のまま残ります
```

セッションは microVM を占有します。リークしたセッションは、タイムアウトするまで課金され続けます。本番環境では、コンテキストマネージャーの使用は必須です。

## VPC モード

エージェントが VPC モードで動作する場合、Browser ツールも VPC 内で実行できます。サブネットとセキュリティグループのパターンは [`vpc.md`](vpc.md) を参照してください（同じサービスリンクロールが Browser の ENI にも適用されます）。VPC 内の Browser から公開インターネットのサイトにアクセスするには NAT ゲートウェイが必要です。パブリックサブネットだけでは、Browser はインターネットにアクセスできません。

## よくある失敗

**セッション開始時の "Access denied":** browser リソース ARN に対する `StartBrowserSession` が IAM に不足しています。`aws sts get-caller-identity` の結果が、ポリシーをアタッチした ID と一致しているか確認してください。

**Strands エージェントでの "Model access denied":** Browser ツール自体は問題ありませんが、エージェントのモデルが有効化されていません。Bedrock コンソール → Model access で、そのリージョンのモデルを有効にしてください。

**Nova Act の API キーに関するエラー:** キーは、提供開始時点では米国の amazon.com アカウント限定です。米国外にいる場合や職場のアカウントを使う場合は、まだ Nova Act を利用できません。Playwright または Strands を使ってください。

**タスクの途中で Browser セッションがタイムアウトする:** デフォルトは 15 分のアイドル時間です。`StartBrowserSession` に `sessionTimeoutSeconds` を渡してください（最大 28800 = 8 時間）。動作の遅いエージェントを隠すためにこれを使わないでください。エージェントを直すか、作業を分割してください。

**ライブビューにセッションが表示されない:** ライブビューには `ConnectBrowserLiveViewStream` の IAM 権限が必要です。また、セッションが `Starting` や `Stopping` ではなく、`Ready` である必要があります。

## 出力

- どのフレームワークが適しているか（Strands、Nova Act、Playwright）
- セッションのライフサイクルを処理した、動作するコード
- browser リソースに絞り込んだ IAM ポリシー
- 必要に応じたオブザーバビリティの設定（ライブビュー、録画）

## 品質基準

- Browser セッションは必ずコンテキストマネージャーか try/finally で囲み、リークさせない
- IAM はアカウント内の `browser/*` に絞り、`Resource: "*"` にしない
- Nova Act の API キーなどのシークレットには、環境変数ではなく `agentcore add credential` と `@requires_api_key` を使う
- エージェントが AgentCore Runtime の外で動作する場合（`.env.local` も認証情報プロバイダーもない場合）も考慮する — 通常は、開発時にはローカルのシークレットを読み、本番では認証情報プロバイダーを使う
