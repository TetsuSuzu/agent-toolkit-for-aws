# 例: カスタマーサポートのエージェント

`agentcore create` でスキャフォールドした、カスタマーサポートのエージェントの、完全で現実的な例です。開発者が「カスタマーサポートのエージェントを作りたい」などのタスク形式のプロンプトを出したときの参考として使います。

## このエージェントが行うこと

製品のポリシー、配送、返品に関する顧客の質問に答えます。フレームワークは Strands、モデルは Bedrock(Claude Sonnet)を使い、Memory もツールもない状態から始めます(どちらも後から追加できます)。

## スキャフォールドのコマンド

```bash
agentcore create \
  --name SupportAgent \
  --framework Strands \
  --protocol HTTP \
  --build CodeZip \
  --model-provider Bedrock \
  --memory none
```

## 生成される `main.py`(注釈付き)

スキャフォールドの後、`app/SupportAgent/main.py` は次のようになります。

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from strands import Agent
from model.load import load_model  # `agentcore create` が model/load.py に生成します

app = BedrockAgentCoreApp()

SYSTEM_PROMPT = """You are a customer support agent for Acme Corp.
You answer questions about product policies, shipping, and returns.

Guidelines:
- Be concise and friendly
- If you don't know the answer, say so — don't make up policies
- For order-specific questions, ask for the order number
- Escalate to a human agent if the customer expresses frustration"""

@app.entrypoint
def invoke(payload, context):
    agent = Agent(
        model=load_model(),
        system_prompt=SYSTEM_PROMPT,
    )
    result = agent(payload.get("prompt", ""))
    return {"response": str(result)}

if __name__ == "__main__":
    app.run()
```

> 生成される `model/load.py` は、クロスリージョン推論プロファイル(例: `global.anthropic.claude-sonnet-4-5-*`)で構成された `BedrockModel` を返します。モデル ID をハードコードする代わりに `load_model()` を使うと、コードは CLI が提供するデフォルトに追従します。別のモデルを使うには、`model/load.py` を編集してください。

## ローカルで試す

```bash
agentcore dev
```

別のターミナルで:

```bash
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What is your return policy?"}'
```

## デプロイする

```bash
agentcore deploy
```

## 自然な次のステップ

基本的なエージェントが動作したあと、開発者は通常、次のいずれかを求めます。

| 「〜したい」 | 次のスキル |
|---|---|
| 「注文をデータベースから調べられるようにしたい」 | `agents-connect`(注文 API 用の Gateway ターゲットを追加する) |
| 「セッションをまたいで顧客の名前を覚えさせたい」 | `agents-build`([`references-ja/memory.md`](../../agents-build/references-ja/memory.md) を読み込む) |
| 「ポリシーに反することを言わないようにしたい」 | `agents-connect`([`references-ja/policy.md`](../../agents-connect/references-ja/policy.md) を読み込む) |
| 「自社のウェブサイトに載せたい」 | `agents-build`([`references-ja/integrate.md`](../../agents-build/references-ja/integrate.md) を読み込む) |
| 「本当に役に立っているか知りたい」 | `agents-optimize` |

## バリエーション

### LangGraph 版

```bash
agentcore create --name SupportAgent --framework LangChain_LangGraph --model-provider Bedrock --memory none
```

生成される `main.py` は、`create_react_agent` と `langchain_aws` を使います。

```python
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.prebuilt import create_react_agent
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from model.load import load_model

app = BedrockAgentCoreApp()
SYSTEM_PROMPT = "..."  # Strands 版と同じ

@app.entrypoint
async def invoke(payload, context):
    graph = create_react_agent(load_model(), tools=[])
    result = await graph.ainvoke({
        "messages": [
            SystemMessage(content=SYSTEM_PROMPT),
            HumanMessage(content=payload["prompt"]),
        ]
    })
    return {"response": result["messages"][-1].content}

if __name__ == "__main__":
    app.run()
```

### OpenAI Agents SDK 版

```bash
agentcore create --name SupportAgent --framework OpenAIAgents --model-provider OpenAI --memory none
```

```python
from agents import Agent, Runner
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
async def invoke(payload, context):
    agent = Agent(
        name="SupportAgent",
        instructions="...",  # Strands 版と同じ
    )
    result = await Runner.run(agent, payload["prompt"])
    return {"response": result.final_output}

if __name__ == "__main__":
    app.run()
```

### Google ADK 版

```bash
agentcore create --name SupportAgent --framework GoogleADK --model-provider Gemini --memory none
```

```python
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

agent = Agent(
    model="gemini-2.5-flash",
    name="SupportAgent",
    description="Customer support agent",
    instruction="...",  # Strands 版と同じ
)

@app.entrypoint
async def invoke(payload, context):
    user_id = payload.get("user_id", "default_user")
    session_id = getattr(context, "session_id", "default_session")
    session_service = InMemorySessionService()
    session = await session_service.create_session(
        app_name="support", user_id=user_id, session_id=session_id
    )
    runner = Runner(agent=agent, app_name="support", session_service=session_service)
    content = types.Content(role="user", parts=[types.Part(text=payload["prompt"])])
    async for event in runner.run_async(user_id=user_id, session_id=session.id, new_message=content):
        if event.is_final_response():
            return {"response": event.content.parts[0].text}

if __name__ == "__main__":
    app.run()
```

### モデルプロバイダーの選択肢

CLI は、4 つのモデルプロバイダーをサポートしています。

| プロバイダー | 適している用途 | 備考 |
|---|---|---|
| `Bedrock` | デフォルト、API キー不要、IAM による認証 | クロスリージョン推論プロファイル(例: `global.anthropic.claude-sonnet-4-5-*`)を使う |
| `Anthropic` | Anthropic の API への直接アクセス | `ANTHROPIC_API_KEY` が必要。`claude-sonnet-4-5-20250929` のようなモデル ID |
| `OpenAI` | GPT-4 / GPT-5 のモデル | `OPENAI_API_KEY` が必要。通常は OpenAI Agents SDK と組み合わせる |
| `Gemini` | Google Gemini のモデル | `GEMINI_API_KEY` が必要。通常は Google ADK と組み合わせる |

コストを重視する用途では、Bedrock の Nova モデル(例: `amazon.nova-micro-v1:0`、`amazon.nova-lite-v1:0`)を検討してください。単純な抽出タスクでは、Claude よりもかなり安価です。モデル選択のガイダンスは、[`agents-optimize/references-ja/cost.md`](../../agents-optimize/references-ja/cost.md) を参照してください。

会話を記憶するチャットボットには、スキャフォールド時に `--memory longAndShortTerm` を追加します。Memory は後から追加することもできます — [`agents-build/references-ja/memory.md`](../../agents-build/references-ja/memory.md) を参照してください。
