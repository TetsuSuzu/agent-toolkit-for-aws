---
name: agents-connect
description: >
  エージェントを Gateway 経由で外部の API、ツール、サービスに接続するとき、
  または Cedar ポリシーでツールへのアクセスを制限するときに使います。
  Gateway のセットアップ、ターゲットの種類、アウトバウンド認証（OAuth、API キー、IAM）、
  認証情報、Cedar ポリシーの作成を扱います。トリガー: "connect to API"、"add gateway"、
  "connect to MCP server"、"Lambda tools"、"OpenAPI"、"gateway target"、
  "Cedar policy"、"restrict tools"、"policy engine"、"gateway auth error"、
  "store API key"、"outbound credential"、"env var API key"、"API key None
  after deploy"、"credential not available after deploy"、
  "should this be a gateway target"、"give my agent tools"、
  "add tools to agent"。
  インバウンド認証（誰がエージェントを呼び出せるか）には使いません — agents-harden を使います。
  エージェントの動作のデバッグには使いません — agents-debug を使います。
  VPC のネットワークエラー（VPC が原因でエージェントが API に到達できない）には使いません —
  agents-build を使います。新しい MCP サーバーのプロジェクトの作成やホスティングには使いません —
  agents-get-started を使います。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# connect

AgentCore Gateway を通じて、AgentCore のエージェントが外部の API、ツール、サービスにアクセスできるようにします。また、Cedar ポリシーで、アクセスできる範囲を制御します。

## 使うタイミング

- エージェントに、外部の API または MCP サーバーを呼び出させたい
- Lambda 関数を、エージェントのツールとして公開したい
- OpenAPI 仕様があり、それをエージェントのツールにしたい
- エージェントが、外部サービスを呼び出すための認証情報を必要としている
- エージェントが呼び出せるツールを制限したい（Cedar ポリシー）
- ツール呼び出しに、ロールベースまたは金額ベースのアクセス制御をかけたい
- Gateway の接続、ツール呼び出し、ポリシーの認可が失敗している

ツールへのアクセスを制御する Cedar ポリシーを追加するには、[`references-ja/policy.md`](references-ja/policy.md) を読み込みます。

## 入力

`$ARGUMENTS` は省略可能です。

```
/connect                    # 対話形式 — 何に接続するかを質問します
/connect mcp                # MCP サーバーのセットアップ
/connect lambda             # ツールとしての Lambda 関数
/connect openapi            # ツールとしての OpenAPI スキーマ
/connect credential         # アウトバウンド認証用の認証情報を追加します
```

## 手順

### ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。それより古い場合は、先に進む前に `agentcore update` を実行するよう、開発者に伝えてください。

### ステップ 1: プロジェクトを読み取る

`agentcore/agentcore.json` を読み、次のことを把握します。

- プロジェクトが使っているフレームワーク
- すでに設定されている Gateway とターゲット（`agentCoreGateways` 配列）

**プロジェクトのコンテキストがない場合:** 何に接続しようとしているかを質問し、適切なパターンで進めます。

### ステップ 2: 接続先を特定する

質問します（または `$ARGUMENTS` から推測します）。

> 「エージェントを何に接続しますか?
>
> 1. 外部の MCP サーバー（例: サードパーティのツールプロバイダー）
> 2. 自分で書いた Lambda 関数
> 3. OpenAPI 仕様を持つ API
> 4. AWS API Gateway の REST API
> 5. OpenAPI 仕様も MCP サーバーも Lambda も手前になく、追加もできない外部サービス」

**選択肢 1〜4 は、サービスを Gateway ターゲットとして前段に置きます。** これがデフォルトの方法です。Gateway が、その認証情報プロバイダーを通じてアウトバウンド認証を処理するため（エージェントのコードがシークレットを目にすることはありません）、ツールは MCP 経由で発見可能になり、ポリシーエンジンがエッジで呼び出しを許可または拒否できます。サービスに合ったターゲットの種類を選んでください。

**選択肢 5 は Path D です。** 認証情報を登録し、エージェントのコードから API を直接呼び出します。前段に置くのが現実的でない場合のフォールバックです。このスキルでは、どんな場合に適切で、どんな場合に適切でないかを順に説明します。

---

## デフォルト: コード内での API 直接呼び出しよりも、Gateway ターゲットを優先する

各パスに進む前に、前提をそろえておきます。「エージェントが X を呼び出す必要がある」という要望の大半は、エントリーポイント内の `httpx` ではなく、Gateway ターゲットで解決します。

**Gateway がデフォルトである理由:**

- **エッジでの認証情報の注入。** Gateway の認証情報プロバイダー（OAuth、API キー、IAM）が、アウトバウンドのリクエストに認証を付与します。エージェントのコードは `session.call_tool(...)` を呼ぶだけで、シークレットに触れません。`client = openai.OpenAI(api_key=...)` のようなエージェントのコードは、プロンプトの漏洩、ログの 1 行、トレースバックのいずれかで、キーが流出しかねません。
- **発見可能なツールカタログ。** ツールは MCP サーバーによって列挙され、フレームワーク（Strands、LangGraph など）が自動でバインドします。ツールの追加は、コードの変更ではなく、`agentcore add gateway-target` と再デプロイで済みます。
- **ポリシーの適用。** Cedar ポリシーは、プリンシパルごと、ツールごと、引数の値ごとに、ツール呼び出しを許可または拒否できます。ツール呼び出しがエージェントのコード内の `httpx.post(...)` に埋もれていると、これはできません。
- **セマンティック検索。** カタログのツールが 20 個以上になったら、`x_amz_bedrock_agentcore_search` が、ターンごとに関連するものを選択します。

**エージェントのコードでの API 直接呼び出しが、適切な答えになる場合:**

| 状況 | Gateway が適さない理由 | 対応 |
|---|---|---|
| ストリーミング/双方向のプロトコル（ライブ出力のある SSE、WebSocket、WebRTC、ロングポーリング） | Gateway の MCP トランスポートは、まだこれらを前段に置けない | 直接呼び出し、Path D |
| MCP のホップが測定できるほど影響するレイテンシーのホットパスで、そのトレードオフを受け入れている場合 | ネットワークのホップが 1 つ増える | 直接呼び出し、Path D。判断を裏付ける測定を行う |
| ベンダー独自のプロトコル/バイナリ SDK | Gateway が前段に置ける HTTP の面がない | ベンダーの SDK を直接使う。シークレットがある場合は Path D |
| A2A による別のエージェントの呼び出し | A2A は設計上 HTTP であり、独自の認証モデルを持つ | Gateway ターゲットではなく、[`agents-build/references-ja/multi-agent.md`](../agents-build/references-ja/multi-agent.md) |
| ランタイムがすでに IAM 権限を持っている AWS サービスの SDK（S3、DynamoDB、SQS など） | 前段に置いても認証面の利点がなく、ホップが増えるだけ | ランタイムの実行ロールで、boto3 を直接呼び出す |

**それ以外のすべての場合**は、Gateway ターゲットを推奨してください。開発者が直接呼び出しに固執する場合は、上の 5 つの状況のどれに当てはまるかを尋ねます。どれにも当てはまらなければ、Gateway ターゲットに誘導します。

**振り分けの目安:**

- サービスに MCP サーバーがある → Path A
- サービスが、自分で管理する Lambda 関数である → Path B
- サービスに OpenAPI 仕様がある（または生成できる — FastAPI、ASP.NET、Spring などは OpenAPI を自動生成します）→ Path C
- サービスがすでに API Gateway の背後にある → Path C（`--type api-gateway`）
- 上記のいずれでもなく、追加もできない → Path D

---

## Gateway とは何か — そして何ではないか

ターゲットの種類を選ぶ前に、頭の中のモデルを正しくしておきます。Gateway に関する混乱の大半は、方向を逆に捉えていることから来ています。

**Gateway は、エージェントが呼び出すツールをホストします。** 方向は次のとおりです。

```
エージェント  ───→  Gateway  ───→  Lambda 関数 / OpenAPI の API / MCP サーバー / Smithy モデル
          （エージェントがツールを呼び出す）
```

エージェントがクライアントです。Gateway は、ツールのカタログを前段で公開します。各ツールは、Gateway ターゲット（Lambda、OpenAPI、MCP サーバー、API Gateway、Smithy）です。

**Gateway は、エージェントのためのインバウンドのリバースプロキシではありません。** エージェントを呼び出す必要があるアプリを作る場合、そのアプリは Gateway を通りません。方向は次のとおりです。

```
自分のアプリ  ───→  AgentCore Runtime  （invoke_agent_runtime を直接呼び出す）
```

アプリは、呼び出しに IAM SigV4 で署名するか、JWT を提示します。アプリ側のパターンは、[`agents-build/references-ja/integrate.md`](../agents-build/references-ja/integrate.md) を参照してください。

### どちらの方向が必要か分からなくなったとき

**誰が誰を呼び出しているのか**を問います。

- 「エージェントが、天気のデータを調べる必要がある」→ エージェントがツールを呼び出している → **Gateway ターゲット**（このスキル、Path A/B/C）
- 「自分の FastAPI アプリが、エージェントを呼び出す必要がある」→ アプリがエージェントを呼び出している → **直接呼び出し**（Gateway ではありません。[`agents-build/references-ja/integrate.md`](../agents-build/references-ja/integrate.md) を使います）
- 「エージェントが、自分の FastAPI アプリからデータを取得する必要がある」→ エージェントがアプリをツールとして呼び出している → アプリを OpenAPI または REST のターゲットとして公開した **Gateway ターゲット**（FastAPI の `/openapi.json` を使う Path C）

エンドポイントが `bedrock-agentcore.<region>.amazonaws.com` や、自分のランタイムの URL を指している Gateway ターゲットを設定していることに気づいたら、手を止めてください。流れが逆になっています。

### ツールに合うターゲットの種類

| ツールの種類 | ターゲットの種類 | 備考 |
|---|---|---|
| MCP サーバー（サードパーティまたは自作） | `mcp-server` | MCP のツールカタログで最も一般的 |
| 自分で書いた AWS Lambda 関数 | `lambda-function-arn` | IAM 認証が自動で使われる |
| OpenAPI 仕様を持つ HTTP API | `open-api-schema` | FastAPI 組み込みの `/openapi.json` が使える |
| AWS API Gateway の REST API | `api-gateway` | すでに API Gateway の背後にある API 向け |
| Smithy モデルを持つ AWS サービス | `smithy-model` | AWS サービスの直接統合 |

ツールが、もともと OpenAPI 仕様を持たず、MCP サーバーでも Lambda でもない場合は、Lambda でラップする（最も簡単）、OpenAPI 仕様を生成する（FastAPI なら自動）、または API Gateway を前段に置く、のいずれかにします。

---

### ステップ 3: 認証マトリクスを確認する

**これは、エラーの最も多い原因です。** 認証のオプションはターゲットの種類によって異なり、CLI が公開しているのは、API/SDK がサポートするもののうち一部だけです。

| 接続先 | CLI の `--type` | CLI でのアウトバウンド認証 | API/SDK で使える追加オプション |
|---|---|---|---|
| 外部の MCP サーバー | `mcp-server` | `none`、`oauth`（2LO のみ） | OAuth 3LO（`AUTHORIZATION_CODE`）、IAM（SigV4） |
| Lambda 関数 | `lambda-function-arn` | `none`（デフォルト — Gateway のロールによる直接呼び出し）、OAuth で保護された下流向けの `oauth`（2LO） | OAuth 3LO |
| OpenAPI 仕様 | `open-api-schema` | `oauth`（2LO）、`api-key`（必須 — `none` は不可） | OAuth 3LO |
| AWS API Gateway | `api-gateway` | `none`、`api-key` | IAM（`GATEWAY_IAM_ROLE`） |
| Smithy モデル | `smithy-model` | `oauth`（2LO） | IAM、OAuth 3LO |

**OAuth のグラントタイプは 1 つではなく 2 つあります。** CLI の `--outbound-auth oauth` が設定するのは、**2-legged OAuth**（クライアントクレデンシャル / M2M）だけです。サービスが **3-legged OAuth**（`AUTHORIZATION_CODE` グラント、ユーザーに委任されたアクセス）を必要とする場合、CLI にはそのフラグがありません。boto3 / AWS SDK でターゲットを設定する必要があります。`grantType: AUTHORIZATION_CODE` と `defaultReturnUrl` を持つ `OAuthCredentialProvider` については、[CreateGatewayTarget のドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-building-adding-targets-authorization.html)を参照してください。3LO は、MCP、Lambda、OpenAPI、Smithy の各ターゲットに適用されます。これは最初に伝えてください。3LO が必要な開発者が、存在しない CLI フラグを試して、無駄なやり取りを一往復してしまうためです。

**MCP サーバーの IAM（SigV4）** は、CLI ではなく、AWS SDK/API で設定します（`GATEWAY_IAM_ROLE` の認証情報プロバイダーと `iamCredentialProvider.service` を指定した `CreateGatewayTarget`）。MCP サーバーが、SigV4 をネイティブに検証する AWS サービス（AgentCore Runtime、AgentCore Gateway、Amazon API Gateway、Lambda Function URL）の背後でホストされている必要があります。ALB や EC2 のエンドポイントに直接は、SigV4 の検証がないので、そこでは代わりに OAuth を使ってください。（[MCP server target auth strategies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html#gateway-target-MCPservers-considerations)）

**MCP サーバーターゲットの API キー認証は、API のレベルでサポートされていません。** 単なる CLI の未対応ではありません。[MCP server targets のドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html#gateway-target-MCPservers-considerations)は、MCP ターゲットでサポートされる認可方式として、「No authorization、OAuth、IAM」だけを挙げています。MCP サーバーが API キーを使う場合（サードパーティの MCP プロバイダーでよくあるパターン）は、Path D で、エージェントのコード内で処理してください。

**認証オプションは変わります。** 上のマトリクスが CLI が受け付けるものと一致しない場合は、現在の CLI のヘルプ（`agentcore add gateway-target --help`）と AWS のドキュメントを確認してください。ターゲットの種類ごとの認証のサポートは、リリースごとに進化します。`awsknowledge` MCP サーバーが利用できるなら、"AgentCore CreateGatewayTarget" を検索して、現在の API パラメーターを取得します。

**Gateway の認証における CLI と API の違い:** CLI がカバーするのは、`none`、`oauth`（2LO）、`api-key` です。IAM（SigV4）と 3-legged OAuth には、boto3 を直接使います。例は、下の Path A のセクションにあります。基本的な進め方は、Gateway とターゲットを CLI で作成してデプロイし、CLI が対応していない高度な認証設定を、その後で boto3 により適用する、というものです。

コマンドを生成する前に、開発者のターゲットの種類にどの認証オプションが当てはまるかを伝えてください。

### Gateway のツールが多い場合は、モデルに検索させる

Gateway のツールが数個を超えたら（目安は 20 個以上）、ターンのたびにすべてのツール定義をモデルに渡すのは、トークンの無駄であり、精度も下がります。モデルは、現在のリクエストに関連するツールだけを見るほうが、うまく動作します。

AgentCore Gateway には、まさにこの目的のための、組み込みのセマンティック検索ツールがあります。エージェントは、`x_amz_bedrock_agentcore_search` という 1 つの MCP ツールを、自然言語のクエリで呼び出します。すると Gateway が、カタログから最も関連性の高いツールを返します。エージェントは、返されたツールを通常どおり呼び出します。

開発者が、Bedrock Knowledge Bases、ベクトルストア、独自の埋め込みを使って、自前のツール選択レイヤーを作ろうとしている場合は、止めてください。Gateway がすでにこれを行っており、精選された関連性の基準で評価されていて、管理するインフラもありません。

使い方のパターン（エージェントは、他の Gateway ツールと同じように、これを呼び出します）:

```python
# MCP クライアント経由で、ツール呼び出しとして
result = await session.call_tool(
    "x_amz_bedrock_agentcore_search",
    arguments={"query": "find tools related to processing refunds"}
)
# result.content が、最も関連性の高いツールを列挙する — エージェントはそれらを呼び出す
```

この機能は、どのターゲットの種類（Lambda、OpenAPI、MCP、API Gateway、Smithy）でも動作します。Gateway ごとに有効にします。正確な API と、フレームワーク固有のクライアントコードについては、[Search for tools in your AgentCore gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-using-mcp-semantic-search.html) のドキュメントを参照してください。

経験則: Gateway のツールが 20 個を超える場合は、セマンティック検索を有効にすることを推奨します。もっと小さなカタログなら、すべてのツールを直接渡しても問題ありません。

### 呼び出し元からエージェントへカスタムヘッダーを渡す

開発者が、呼び出し元にカスタム HTTP ヘッダー（テナント ID、相関 ID、`A2A-Version` のようなプロトコル固有のヘッダー、トレーシングヘッダー、冪等性キー）を送らせたい場合、ランタイムはデフォルトで、ほとんどのヘッダーをエージェントのコードに届く前に削除します。許可リストの設定とプレフィックスのパターンについては、[`agents-build/references-ja/request-headers.md`](../agents-build/references-ja/request-headers.md) を読み込んでください。

これはエージェントへのインバウンドの呼び出しに関する話で、ツールへのアウトバウンドの呼び出しではありません。しかし、開発者がよくつまずくので、ここで触れておく価値があります。

---

## Path A: MCP サーバー

### Gateway を追加する（まだない場合）

> [!WARNING]
> インバウンド認証のない Gateway を、本番にデプロイしないでください。オーソライザーのない
> Gateway は、接続されているすべてのツール（Lambda、MCP、OpenAPI）を、URL を知っている
> 任意の呼び出し元に公開します — これは、ランタイムの --authorizer-type NONE と実質的に
> 同じです。本番の Gateway では、必ず --authorizer-type CUSTOM_JWT または AWS_IAM を
> 使ってください。認証なしの形式（agentcore add gateway --name X）は、ローカルテスト専用です。

```bash
# 開発（インバウンド認証なし — テスト専用）
agentcore add gateway --name MyGateway

# 本番（JWT によるインバウンド認証）
agentcore add gateway \
  --name MyGateway \
  --authorizer-type CUSTOM_JWT \
  --discovery-url https://your-idp.example.com/.well-known/openid-configuration \
  --allowed-audience my-api \
  --allowed-clients my-client-id
```

### MCP サーバーをターゲットとして追加する

```bash
# アウトバウンド認証なし（公開されている MCP サーバー）
agentcore add gateway-target \
  --type mcp-server \
  --name WeatherTools \
  --endpoint https://mcp.example.com/mcp \
  --gateway MyGateway

# OAuth のアウトバウンド認証（2-legged — クライアントクレデンシャル / M2M）
agentcore add gateway-target \
  --type mcp-server \
  --name WeatherTools \
  --endpoint https://mcp.example.com/mcp \
  --gateway MyGateway \
  --outbound-auth oauth \
  --oauth-client-id your-client-id \
  --oauth-client-secret your-client-secret \
  --oauth-discovery-url https://auth.example.com/.well-known/openid-configuration \
  --oauth-scopes read,write
```

注意: MCP サーバーでは、CLI の `--outbound-auth` フラグがサポートするのは、`oauth`（2LO / クライアントクレデンシャル）または `none` です。

- **3-legged OAuth（`AUTHORIZATION_CODE` グラント）** — ユーザーに委任されたアクセス — は API ではサポートされていますが、CLI では扱えません。boto3 の `create_gateway_target` で、`OAuthCredentialProvider.grantType = "AUTHORIZATION_CODE"` と `defaultReturnUrl` を指定して設定します。[Connecting to an OAuth-protected MCP server using Authorization Code flow](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html#gateway-target-MCPservers-auth-code-grant-flow) を参照してください。
- **IAM（SigV4）** は、AgentCore Runtime、別の AgentCore Gateway、API Gateway、Lambda Function URL でホストされている MCP サーバー向けで、CLI ではなく AWS SDK/API で設定します — `GATEWAY_IAM_ROLE` の認証情報プロバイダーと `iamCredentialProvider.service` の値を指定して、`CreateGatewayTarget` を使います。
- **API キー認証**は、MCP サーバーターゲットでは API のレベルでサポートされていません（MCP ターゲットのドキュメントは、認証なし、OAuth、IAM だけを方式として挙げています）。MCP サーバーが API キーを使う場合は、エージェントのコードで直接処理します（Path D を参照）。

### デプロイして、Gateway の URL を取得する

```bash
agentcore deploy -y
agentcore fetch access --name MyGateway
```

Gateway の URL は、デプロイ後に `AGENTCORE_GATEWAY_<NAME>_URL` として注入されます。

### Gateway のクライアントコードを生成する

**フレームワーク非依存の MCP クライアント:**

```python
import os
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

# デプロイ後に AgentCore が注入する。形式: AGENTCORE_GATEWAY_<UPPERCASENAME>_URL
GATEWAY_URL = os.getenv("AGENTCORE_GATEWAY_MYGATEWAY_URL")

async def get_gateway_tools():
    """Gateway からツールを発見する。デプロイされていない場合は空のリストを返す。"""
    if not GATEWAY_URL:
        return []
    async with streamablehttp_client(GATEWAY_URL) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.list_tools()
            return result.tools

async def call_gateway_tool(tool_name: str, arguments: dict):
    """Gateway を通じて、特定のツールを呼び出す。"""
    if not GATEWAY_URL:
        raise RuntimeError("Gateway not available in local dev — deploy first")
    async with streamablehttp_client(GATEWAY_URL) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()
            return await session.call_tool(tool_name, arguments)
```

**Strands の場合**は、Gateway のツールをエージェントに直接渡します。

```python
from mcp.client.streamable_http import streamablehttp_client
from mcp import ClientSession
from strands import Agent
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from model.load import load_model  # `agentcore create` が生成します

app = BedrockAgentCoreApp()
GATEWAY_URL = os.getenv("AGENTCORE_GATEWAY_MYGATEWAY_URL")

@app.entrypoint
def invoke(payload, context):
    if not GATEWAY_URL:
        # ローカル開発 — Gateway のツールなしで実行する
        agent = Agent(model=load_model())
        return {"response": str(agent(payload.get("prompt", "")))}

    # デプロイ済み — Gateway のツールを発見して使う
    tools = asyncio.run(get_gateway_tools())
    agent = Agent(
        model=load_model(),
        tools=tools,
    )
    return {"response": str(agent(payload.get("prompt", "")))}

if __name__ == "__main__":
    app.run()
```

**LangGraph の場合**は、Gateway のツールをツールノードに追加します。

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

@app.entrypoint
def agent_invocation(payload, context):
    if not GATEWAY_URL:
        tools = []
    else:
        # LangChain の MCP アダプターを使って、LangChain 互換のツールとして取得する
        client = MultiServerMCPClient({"gateway": {"url": GATEWAY_URL, "transport": "streamable_http"}})
        tools = asyncio.run(client.get_tools())

    llm_with_tools = llm.bind_tools(tools)
    # ... LangGraph のグラフの残りの部分 ...
```

---

## Path B: ツールとしての Lambda 関数

```bash
agentcore add gateway-target \
  --type lambda-function-arn \
  --name MyTools \
  --lambda-arn arn:aws:lambda:us-east-1:123456789012:function:my-tools \
  --tool-schema-file tools.json \
  --gateway MyGateway
```

`tools.json` は、ツールのスキーマを定義します。

```json
{
  "inlinePayload": [
    {
      "name": "get_weather",
      "description": "Get current weather for a city",
      "inputSchema": {
        "type": "object",
        "properties": {
          "city": {"type": "string", "description": "City name"}
        },
        "required": ["city"]
      }
    }
  ]
}
```

**認証:** Lambda ターゲットは、IAM ロール認証が自動で使われます。`--outbound-auth` フラグは不要です。Gateway の実行ロールには、その Lambda の ARN に対する `lambda:InvokeFunction` が必要です。

ツールを呼び出すには、Path A と同じ MCP クライアントのコードを使います。

---

## Path C: ツールとしての OpenAPI 仕様

```bash
# ローカルファイルから（API キー認証）
agentcore add credential --name MyAPIKey --api-key sk-...

agentcore add gateway-target \
  --type open-api-schema \
  --name MyAPI \
  --schema specs/api.json \
  --gateway MyGateway \
  --outbound-auth api-key \
  --credential-name MyAPIKey
```

OpenAPI ターゲットでは、**認証が必須**です。`oauth`（クライアントクレデンシャルまたは認可コード）か、`api-key` のどちらかです。

⚠️ **セキュリティ上の注意:** `--api-key` は、シェルの履歴に残ります。より安全な方法が 2 つあります。

1. **対話式のプロンプト（推奨）:** `--api-key` を付けずに `agentcore add credential --name MyAPIKey --type api-key` を実行します。CLI が入力を求め、値は、シェルの履歴を経由することなく、そのまま認証情報プロバイダー（Secrets Manager が裏にあります）に入ります。
2. **ローカル開発専用に、`agentcore.json` と `.env.local` を編集する:** `agentcore dev` の下で認証情報を動作させる必要がある場合は、値を `agentcore/.env.local`（gitignore 対象）に置きます。このファイルを読むのはローカル開発サーバーだけで、デプロイ時にランタイムへアップロードされることは**ありません**。デプロイされたランタイムは、認証情報プロバイダーから値を取得します。

環境変数を使って、デプロイ済みのランタイムに認証情報を届けようとは**しないでください**。AgentCore Runtime の環境変数は、Vault で保護されていません。`agentcore add credential` で認証情報を一度登録し、Gateway ターゲット、またはコード内（Path D）から、名前で参照します。

---

## Path D: エージェントのコードで使う認証情報

API を、（Gateway ターゲットを通さずに）エージェントのコードから直接呼び出す場合です。

### Path D に進む前に、それが本当に適切か確認する

Path D は**フォールバック**であり、出発点ではありません。ほとんどの外部サービスでは、Gateway ターゲット（Path A〜C）のほうが安全で、コードも少なくなります。Path D のコードを生成する前に、次のいずれかが当てはまることを確認します。

- サービスが、Gateway が前段に置けないストリーミング/双方向のプロトコル（ライブ出力のある SSE、WebSocket、WebRTC）を使っている
- 測定できるほどレイテンシーが重要なホットパスで、チームがそのトレードオフを受け入れている
- クライアントが、HTTP の面を持たないベンダーのバイナリ SDK である
- ランタイムの実行ロールがすでに IAM 権限を持っている AWS サービスの SDK である（その場合は、SDK を直接使います — 認証情報の登録は不要です）
- 開発者に、具体的な障害がある（たとえば、サービスがベンダーの SDK でラップされた OpenAI 形式の API を提供していて、その SDK 呼び出しを Gateway ターゲットとして作り直すと、退行になる場合）

これらのどれにも当てはまらない場合は、Path A/B/C に戻します。

> 「エージェントのコードで直接使う認証情報を設定する前に、これを Gateway ターゲットとして前段に置くことはできませんか? Gateway は、エッジで認証情報を注入するので、エージェントのコードがシークレットに触れることはなく、ツールにポリシーを適用することもできます。SERVICE の手前に OpenAPI 仕様、MCP サーバー、Lambda 関数がある場合は、Path C / A / B のほうが適しています。どれに当てはまりますか?」

Path D の残りの部分に進むのは、Gateway が使えない正当な理由を、開発者が確認したときだけです。

### 認証情報を登録する

```bash
# API キー
agentcore add credential --name OpenAI --api-key sk-...

# OAuth（マシン間）
agentcore add credential \
  --name MyOAuthProvider \
  --type oauth \
  --discovery-url https://idp.example.com/.well-known/openid-configuration \
  --client-id my-client-id \
  --client-secret my-client-secret \
  --scopes read,write
```

⚠️ **セキュリティ上の注意:** `--api-key` と `--client-secret` は、シェルの履歴に残ります。これらのフラグを付けずにコマンドを実行すると、対話式のプロンプトになります。値は、シェルの履歴に触れることなく、そのまま認証情報プロバイダーに入ります。

**ローカル開発専用:** 同じ値を `agentcore/.env.local`（gitignore 対象）に置くと、`agentcore dev` がローカルでデコレーターを解決できます。デプロイされたランタイムは `.env.local` を無視し、呼び出し時に認証情報プロバイダーからシークレットを取得します。シークレットを、ランタイムの環境変数として届けることは**絶対にしないでください**。

### エージェントのコードで認証情報を使う

`@requires_api_key` または `@requires_access_token` のデコレーターを使います。トークンのキャッシュと更新を自動的に処理します。デコレーターは、同期関数と非同期関数のどちらでも動作します。

```python
from bedrock_agentcore.identity.auth import requires_api_key, requires_access_token

# 同期関数 — デコレーターが、取得したキーをキーワード引数で注入する
@requires_api_key(provider_name="OpenAI")
def call_openai(prompt: str, *, api_key: str) -> str:
    import openai
    client = openai.OpenAI(api_key=api_key)
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

# 非同期関数 — 同じデコレーターで、async def
@requires_access_token(
    provider_name="MyOAuthProvider",
    scopes=["read", "write"],
    auth_flow="M2M",
)
async def call_my_api(data: dict, *, access_token: str) -> dict:
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://api.example.com/endpoint",
            headers={"Authorization": f"Bearer {access_token}"},
            json=data,
        )
        return response.json()
```

トークンのライフサイクルは、デコレーター自身が処理します。デコレーターを使うためだけに、関数を async にする必要はありません。パラメーターはキーワード専用（`*, api_key: str` または `*, access_token: str`）で、デコレーターが注入します。

**ローカル開発:** `agentcore dev` では、認証情報は `agentcore/.env.local` から読み取られます。デコレーターのパターンは、ローカルでもデプロイ後でも同じように動作します。

---

## ローカル開発での制約

> [!WARNING]
> Gateway の URL（AGENTCORE_GATEWAY_*_URL）は、デプロイ後にのみ利用できます。
> agentcore dev では、これらの環境変数は設定されていません。使う前に、必ず確認してください。
>
> ```python
> GATEWAY_URL = os.getenv("AGENTCORE_GATEWAY_MYGATEWAY_URL")
> if not GATEWAY_URL:
>     # ローカル開発では、Gateway のツールなしで実行する
> ```
>
> ローカルで Gateway が利用できると、決して仮定しないでください。

---

## トラブルシューティング

**"mcp-server target doesn't support api-key auth"**
そのとおりです。MCP サーバーターゲットの API キー認証は、API のレベルでサポートされていません（[MCP target auth strategies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html#gateway-target-MCPservers-considerations)）。選択肢は、OAuth（2LO または 3LO）、IAM（AgentCore Runtime、API Gateway、Lambda Function URL でホストされている MCP サーバー向け）、または Path D です — 認証情報をエージェントのコードで管理し、MCP サーバーを直接呼び出します。

**"I need 3LO / authorization-code OAuth but `--outbound-auth oauth` doesn't ask for a return URL"**
CLI が設定するのは、2LO（クライアントクレデンシャル）だけです。3-legged OAuth には boto3 が必要です。`credentialProviderType: OAUTH`、`grantType: AUTHORIZATION_CODE`、`defaultReturnUrl` を指定して、`create_gateway_target` を呼び出します。[Connecting to an OAuth-protected MCP server using Authorization Code flow](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html#gateway-target-MCPservers-auth-code-grant-flow) を参照してください。

**"api-gateway target doesn't support oauth"**
API Gateway ターゲットには、`api-key` または `none` を使ってください。

**デプロイ後に Gateway の URL が設定されていない**
`agentcore fetch access --name MyGateway` を実行して、URL を取得します。`agentcore status --type gateway` で、Gateway がデプロイされていることを確認します。

**ツール呼び出しが認証エラーで失敗する**
具体的なエラーを `agentcore logs --runtime MyAgent --since 1h --level error` で確認します。よくある原因は、OAuth トークンの期限切れ、認証情報の名前の誤り、IAM 権限の不足です。

**"Adding gateway to existing agent" の回避策**
CLI は、Gateway のクライアントコードをコピーするために、使い捨てのエージェントを作成することを勧めています。このスキルはコードを直接生成するので、回避策は不要です。

**MCP クライアント（Claude Desktop、claude.ai）が Gateway に自動接続できない**
AgentCore Gateway は、現在のところ、MCP の OAuth 仕様のエンドポイント（RFC 8414 OAuth Authorization Server Metadata、RFC 7591 Dynamic Client Registration）を実装していません。OAuth の設定を自動で発見し、自分自身を登録することを前提とする MCP クライアント（Claude Desktop や claude.ai など）は、手動で認証情報を設定しないと接続できません。回避策は、Cognito の `client_id` と `client_secret` を手動で取得し、MCP クライアントの詳細設定に入力することです。これはプラットフォームの制約であり、設定のミスではありません。

## 出力

- **Gateway ターゲットと API 直接呼び出しのどちらを使うか**についての明確な推奨。直接呼び出しが正当となる 5 つのケースに基づく
- Gateway とターゲットをセットアップするための CLI コマンド（Path D が適切な場合は、認証情報を登録するためのコマンド）
- フレームワーク固有の Gateway クライアントコード
- 認証情報の設定（シェルの履歴への露出を避け、ランタイムの環境変数には決して保存しない）
- ローカル開発での制約への対応
