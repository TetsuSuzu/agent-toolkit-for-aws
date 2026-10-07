# multi-agent

エージェントが他のエージェントに作業を委任する、AgentCore のシステムを構築します。

## 使うタイミング

- オーケストレーターのエージェントが、複雑なタスクをスペシャリストに委任するようにしたい
- エージェントがそれぞれ異なる役割と機能を持つシステムを構築している
- エージェント同士が、A2A 標準を通じて互いを発見し、通信できるようにしたい
- 複数のエージェントで同じ Memory を共有したい

## 入力

`$ARGUMENTS` は省略可能です。

```
/multi-agent              # 対話形式 — どのパターンが必要かを質問します
/multi-agent a2a          # A2A プロトコルのセットアップ
/multi-agent direct       # 直接呼び出しパターン
/multi-agent memory       # エージェント間で共有する Memory
```

## パターンを選ぶ

### ステップ 1: スペシャリストのエージェントをデプロイする

スペシャリストは、標準的な AgentCore のエージェントです。通常どおりデプロイします。

```bash
agentcore create --name SpecialistAgent --defaults
# ... app/SpecialistAgent/main.py にスペシャリストのロジックを追加する ...
agentcore deploy -y
```

デプロイ後に、スペシャリストのランタイム ARN を取得します。

```bash
agentcore status --runtime SpecialistAgent --json | jq -r '.runtimes[0].arn'
```

### ステップ 2: スペシャリストをオーケストレーターのツールとして追加する

オーケストレーターは、`bedrock-agentcore:InvokeAgentRuntime` を通じてスペシャリストを呼び出します。このツールを、オーケストレーターのエージェントコードに追加します。

```python
import os
import json
import boto3
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

# この環境変数は、オーケストレーターのデプロイ設定で指定する
SPECIALIST_ARN = os.getenv("SPECIALIST_AGENT_ARN")
REGION = os.getenv("AWS_REGION", "us-east-1")

def call_specialist(prompt: str, session_id: str = None) -> str:
    """
    スペシャリストのエージェントを呼び出し、そのレスポンスを返す。
    スペシャリストは、独自の分離されたセッションで実行される。
    """
    client = boto3.client("bedrock-agentcore", region_name=REGION)

    kwargs = {
        "agentRuntimeArn": SPECIALIST_ARN,
        "qualifier": "DEFAULT",  # または、固定する特定のバージョン番号
        "payload": json.dumps({"prompt": prompt}).encode(),
    }
    if session_id:
        kwargs["runtimeSessionId"] = session_id

    response = client.invoke_agent_runtime(**kwargs)
    # response["response"] は StreamingBody — 読み込んでから JSON をパースする
    body = response["response"].read()
    result = json.loads(body.decode() if isinstance(body, bytes) else body)
    return result.get("response", result.get("result", str(result)))
```

qualifier に `"DEFAULT"` を渡すと、ライブバージョンが呼び出されます。特定のバージョンに固定する場合（ステージング用の固定、カナリア、ロールバック）は、代わりに数値のバージョン文字列を渡します。完全なワークフローは [`agents-deploy/references/versioning.md`](../../agents-deploy/references/versioning.md) を参照してください。

**Strands の場合**は、`@tool` として登録します。

```python
from strands import Agent, tool

@tool
def delegate_to_specialist(task: str) -> str:
    """
    複雑な分析タスクをスペシャリストのエージェントに委任する。
    タスクに深い専門知識が必要な場合に使用する。

    Args:
        task: スペシャリストに依頼する具体的なタスクまたは質問。

    Returns:
        スペシャリストの詳細な回答。
    """
    return call_specialist(task)

@app.entrypoint
def invoke(payload, context):
    agent = Agent(
        model=load_model(),  # `agentcore create` が生成します
        system_prompt="""You are an orchestrator. For complex analysis tasks,
        delegate to the specialist using the delegate_to_specialist tool.
        Synthesize the specialist's response for the user.""",
        tools=[delegate_to_specialist],
    )
    result = agent(payload.get("prompt", ""))
    return {"response": str(result)}

if __name__ == "__main__":
    app.run()
```

**LangGraph の場合**は、ツールノードとして追加します。

```python
from langchain_core.tools import tool as lc_tool

@lc_tool
def delegate_to_specialist(task: str) -> str:
    """Delegate complex tasks to the specialist agent."""
    return call_specialist(task)

# LangGraph のツールノードに追加する
tools = [delegate_to_specialist]
tool_node = ToolNode(tools)
llm_with_tools = llm.bind_tools(tools)
```

**OpenAI Agents SDK の場合**は、`@function_tool` として登録します。

```python
from agents import Agent, Runner, function_tool

@function_tool
def delegate_to_specialist(task: str) -> str:
    """Delegate a complex analysis task to the specialist agent.
    Use when the task requires deep domain expertise."""
    return call_specialist(task)

@app.entrypoint
async def invoke(payload, context):
    agent = Agent(
        name="Orchestrator",
        instructions="For complex analysis, delegate to the specialist using delegate_to_specialist. Synthesize the response for the user.",
        tools=[delegate_to_specialist],
    )
    result = await Runner.run(agent, payload["prompt"])
    return {"response": result.final_output}
```

**Google ADK の場合**は、エージェントの `tools=[]` リストに通常の関数として渡します。注意: 公式のサンプルでは、ADK のマルチエージェントパターンに A2A を使っています（`awslabs/agentcore-samples/02-use-cases/A2A-multi-agent-incident-response/host_adk_agent/` を参照）。以下の直接呼び出しパターンは、ADK のベーステンプレートから類推したものです。本番で頼る前に、ご利用の ADK バージョンで検証してください。

```python
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

def delegate_to_specialist(task: str) -> str:
    """Delegate complex analysis to the specialist agent."""
    return call_specialist(task)

agent = Agent(
    model="gemini-2.5-flash",
    name="orchestrator",
    description="Orchestrator that delegates complex tasks to specialists.",
    instruction="For complex analysis, call delegate_to_specialist and synthesize the response.",
    tools=[delegate_to_specialist],
)

@app.entrypoint
async def invoke(payload, context):
    user_id = payload.get("user_id", "default_user")
    session_id = getattr(context, "session_id", "default_session")
    session_service = InMemorySessionService()
    session = await session_service.create_session(
        app_name="orchestrator", user_id=user_id, session_id=session_id
    )
    runner = Runner(agent=agent, app_name="orchestrator", session_service=session_service)
    content = types.Content(role="user", parts=[types.Part(text=payload["prompt"])])
    async for event in runner.run_async(user_id=user_id, session_id=session.id, new_message=content):
        if event.is_final_response():
            return {"response": event.content.parts[0].text}
```

検証済みの ADK マルチエージェントパターンが必要な場合は、直接呼び出しではなく A2A を使用してください。以下の A2A のセクションと、上記のサンプルを参照してください。

**Claude Agent SDK の場合:** 公式のサブエージェントパターンについては、[`awslabs/agentcore-samples/03-integrations/agentic-frameworks/claude-agent/claude-sub-agents/`](https://github.com/awslabs/agentcore-samples/tree/main/03-integrations/agentic-frameworks/claude-agent/claude-sub-agents) を参照してください。このプラグインには Claude SDK の委任パターンを含めていません。このサンプルのほうが、私たちが類推できるどんなものよりも新しいためです。

### ステップ 3: IAM 権限を付与する

オーケストレーターの実行ロールには、スペシャリストを呼び出す権限が必要です。

```json
{
  "Effect": "Allow",
  "Action": "bedrock-agentcore:InvokeAgentRuntime",
  "Resource": "arn:aws:bedrock-agentcore:<REGION>:<YOUR_ACCOUNT_ID>:runtime/SpecialistAgent-*"
}
```

これを、`agentcore/agentcore.json` のオーケストレーターのエージェントの IAM 設定に追加するか、デプロイ後に、自動作成された実行ロールへ手動で追加します。

### ステップ 4: デプロイ時にスペシャリストの ARN を渡す

スペシャリストの ARN を、オーケストレーターのデプロイにおける環境変数として追加します。

```bash
# スペシャリストをデプロイしたあと、その ARN を取得する:
SPECIALIST_ARN=$(agentcore status --runtime SpecialistAgent --json | jq -r '.runtimes[0].arn')

# ローカル開発では、.env.local に書き込む:
echo "SPECIALIST_AGENT_ARN=$SPECIALIST_ARN" >> agentcore/.env.local
```

**デプロイ済みのオーケストレーター**では、スペシャリストの ARN を環境変数として利用できる必要があります。推奨されるパターンは次のとおりです。

1. **`agentcore/agentcore.json` を編集する** — オーケストレーターのエージェントのエントリを見つけて、その設定に環境変数を追加します（正確なフィールド名は CLI のバージョンによって異なります。編集後に `agentcore validate` を実行してください）。現行の CLI バージョンでは、エージェントの環境変数は通常、デプロイ設定を通じて管理されます。

2. **または CDK のオーバーライドを使う** — CDK のコンストラクトを直接使っているチームは、Runtime コンストラクトの environment プロパティに環境変数を設定します。

3. **またはデプロイ時に環境変数を書き込む** — チームによっては、`agentcore/.env.local` と `agentcore/agentcore.json` の更新をまとめて生成する、デプロイ前のスクリプトを使っています。

```bash
# pre-deploy.sh — オーケストレーターをデプロイするたびに、事前に実行する
SPECIALIST_ARN=$(agentcore status --runtime SpecialistAgent --json | jq -r '.runtimes[0].arn')
echo "SPECIALIST_AGENT_ARN=$SPECIALIST_ARN" >> agentcore/.env.local

# その後デプロイする
agentcore deploy -y
```

現時点では、CLI の `agentcore add agent` に専用の `--env` フラグはありません。お使いの CLI バージョンで使えるオプションは、`agentcore add agent --help` で確認してください。

---

## パターン 2: A2A プロトコル

スペシャリストは A2A 標準を公開します。エージェントカードで発見でき、JSON-RPC で呼び出せます。AgentCore の A2A ランタイムが、HTTP サーバー、ポートのバインド（9000）、エージェントカードの配信を代わりに処理します。

### ステップ 1: A2A スペシャリストを構築する

`bedrock-agentcore` の `serve_a2a` ヘルパーを使います。これは、`agentcore create --protocol A2A` で CLI が生成するものと一致します。

```python
# app/SpecialistA2A/main.py
from strands import Agent, tool
from strands.multiagent.a2a.executor import StrandsA2AExecutor
from bedrock_agentcore.runtime import serve_a2a
from model.load import load_model


@tool
def analyze_data(dataset_name: str) -> str:
    """指定された名前のデータセットに対して、詳細な分析を実行する。"""
    # ここにスペシャリストのロジックを書く
    return f"Analysis results for {dataset_name}..."


agent = Agent(
    model=load_model(),
    system_prompt="You are an analysis specialist. Use tools when appropriate.",
    tools=[analyze_data],
)

if __name__ == "__main__":
    serve_a2a(StrandsA2AExecutor(agent))
```

```
# requirements.txt
strands-agents[a2a]
bedrock-agentcore
```

`serve_a2a` は、ポート 9000 のバインド、`/.well-known/agent-card.json` でのエージェントカードの生成、JSON-RPC のルーティングを自動的に処理します。FastAPI や uvicorn は不要です。

### ステップ 2: A2A スペシャリストをデプロイする

```bash
agentcore create --name SpecialistA2A --protocol A2A
# CLI は、上に示した serve_a2a パターンで app/SpecialistA2A/main.py を生成する — スペシャリストのロジックに合わせてカスタマイズする
agentcore deploy -y
```

デプロイ後に、ランタイムの URL を取得します。

```bash
agentcore fetch access --name SpecialistA2A --type agent
```

### ステップ 3: ローカルでテストする

```bash
# A2A サーバーをローカルで起動する（プロジェクトのディレクトリから）
agentcore dev

# エージェントカード（ディスカバリー）をテストする
curl http://localhost:9000/.well-known/agent-card.json | jq .

# メッセージを送信する
curl -X POST http://localhost:9000 \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": "req-001",
    "method": "message/send",
    "params": {
      "message": {
        "role": "user",
        "parts": [{"kind": "text", "text": "What is 42 * 17?"}],
        "messageId": "msg-001"
      }
    }
  }' | jq .
```

### ステップ 4: オーケストレーターから A2A スペシャリストを呼び出す

スペシャリストの URL は、秘密ではない識別子なので、オーケストレーターのデプロイ設定の環境変数で渡します。Bearer トークンは**秘密情報**です。デプロイ済みのランタイムでは、`os.getenv(...)` に入れて保持**しないでください**（ランタイムの環境変数は、Vault で保護されていません）。OAuth の M2M プロバイダーを一度登録し、その後は `@requires_access_token` を使って、呼び出し時に新しいトークンを取得します。

```bash
# 一度だけ実行: スペシャリスト用のトークンを発行する OAuth プロバイダーを登録する。
# --client-secret を省略すると、対話的な入力になる（値は認証情報プロバイダーに直接保存される）。
agentcore add credential \
  --name SpecialistA2A \
  --type oauth \
  --discovery-url https://<YOUR_IDP>/.well-known/openid-configuration \
  --client-id <CLIENT_ID> \
  --scopes a2a.invoke
```

```python
import asyncio
import os
from uuid import uuid4
import httpx
from a2a.client import A2ACardResolver, ClientConfig, ClientFactory
from a2a.types import Message, Part, Role, TextPart
from bedrock_agentcore.identity.auth import requires_access_token

# 秘密ではない識別子 — 環境変数から取得してよい。
SPECIALIST_URL = os.getenv("SPECIALIST_A2A_URL")

@requires_access_token(
    provider_name="SpecialistA2A",
    scopes=["a2a.invoke"],
    auth_flow="M2M",
)
async def call_a2a_specialist(message: str, *, access_token: str) -> str:
    session_id = str(uuid4())
    headers = {
        "Authorization": f"Bearer {access_token}",
        "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id": session_id,
    }

    async with httpx.AsyncClient(timeout=300, headers=headers) as http_client:
        resolver = A2ACardResolver(httpx_client=http_client, base_url=SPECIALIST_URL)
        agent_card = await resolver.get_agent_card()

        config = ClientConfig(httpx_client=http_client, streaming=False)
        client = ClientFactory(config).create(agent_card)

        msg = Message(
            kind="message",
            role=Role.user,
            parts=[Part(TextPart(kind="text", text=message))],
            message_id=uuid4().hex,
        )

        async for event in client.send_message(msg):
            if hasattr(event, "parts"):
                return " ".join(p.text for p in event.parts if hasattr(p, "text"))
    return ""

# オーケストレーターのエントリーポイントで使う:
@app.entrypoint
def invoke(payload, context):
    result = asyncio.run(call_a2a_specialist(payload.get("prompt", "")))
    return {"response": result}
```

デコレーターが、キャッシュと更新を処理します。ローカル開発では、`agentcore dev` がデコレーターを解決できるように、OAuth の値を `agentcore/.env.local` に置きます。デプロイ済みのランタイムは、代わりに認証情報プロバイダーからそれらを読み取ります。

---

## エージェント間で共有する Memory

Memory は最上位のリソースであり、単一のエージェントの配下にはありません。複数のエージェントが、同じ環境変数を読むことで共有できます。

### セットアップ

1. 共有する Memory リソースを 1 つ作成します。

```bash
agentcore add memory --name SharedMemory --strategies SEMANTIC,USER_PREFERENCE
```

1. 各エージェントのコードで、同じ環境変数を読みます。

```python
MEMORY_ID = os.getenv("MEMORY_SHAREDMEMORY_ID")
```

1. 一貫した `actor_id` の方式を使います（通常はエンドユーザーの ID）。これにより、両方のエージェントが、同じユーザーの Memory を読み書きします。

### 重要な考慮事項

複数のエージェントが Memory を共有すると、同じ名前空間を共有することになります。衝突を避けるため、名前空間付きのパスを使ってください。

```python
# オーケストレーターは /orchestrator/ 名前空間に書き込む
memory_client.create_event(
    memory_id=MEMORY_ID,
    actor_id=user_id,
    session_id=session_id,
    messages=[("User asked about X", "user")],
)

# スペシャリストはすべての名前空間から読み取る
turns = memory_client.get_last_k_turns(
    memory_id=MEMORY_ID,
    actor_id=user_id,
    session_id=session_id,
    k=5,
)
```

---

## トラブルシューティング

**A2A サーバーが応答しない:**

- ポート 9000 で動作していることを確認する（8080 ではない）
- エージェントカードのエンドポイントが応答することを確認する: `curl http://localhost:9000/.well-known/agent-card.json`
- `main.py` が `serve_a2a(StrandsA2AExecutor(agent))` を使っていることを確認する — 以前の `A2AServer + FastAPI` パターンは非推奨で、こちらに置き換えられている

**直接呼び出しで権限が拒否される:**

- オーケストレーターの実行ロールに `bedrock-agentcore:InvokeAgentRuntime` があることを確認する
- リソース ARN のパターンが、スペシャリストの ARN と一致していることを確認する
- IAM の変更が反映されるまで約 30 秒かかる

**スペシャリストが見つからない:**

- 環境変数 `SPECIALIST_AGENT_ARN` が正しく設定されていることを確認する
- `agentcore status --runtime SpecialistAgent` が `deployed` 状態を示していることを確認する

**A2A の認証エラー:**

- A2A は SigV4 と OAuth 2.0 をサポートしている — 正しい認証方式を使っているか確認する
- 正しい Bearer トークンを取得する: `agentcore fetch access --name SpecialistA2A --type agent`

## 出力

- 適切なパターンを選ぶための判断ツリー
- 選択したパターンの完全なコード（オーケストレーター + スペシャリスト）
- エージェント間の呼び出しのための IAM ポリシー
- ローカルでのテスト用コマンド

## 品質基準

- パターンの推奨が、開発者のレイテンシーと相互運用性の要件に合っている
- 生成されたコードに、エージェント間の呼び出しに必要な正しい IAM 権限が含まれている
- A2A サーバーが、`serve_a2a(StrandsA2AExecutor(agent))` を使って、ポート 9000（8080 ではない）で動作する
- エージェントカードが `/.well-known/agent-card.json` にあり、機能が正しく記載されている
- 共有 Memory で、エージェント間で一貫した `actor_id` の方式を使っている
