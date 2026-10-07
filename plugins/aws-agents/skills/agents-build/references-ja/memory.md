# memory

AgentCore Memory の追加、設定、デバッグを行います。AgentCore Memory は、エージェントがセッションをまたいで物事を記憶できるようにするマネージドサービスです。

## 使うタイミング

- エージェントに、別々のセッションをまたいで、ユーザーの好み、事実、会話履歴を記憶させたい
- `agentcore create` または `agentcore add memory` で Memory を追加したので、エージェントのコードに組み込む必要がある
- Memory の想起が期待どおりに動作しない
- 複数のエージェントで Memory を共有したい

セッション内の会話履歴には、このスキルを使わないでください。それはランタイムが自動的に処理するため、設定は不要です。

## 入力

`$ARGUMENTS` は省略可能です。指定された場合は、Memory リソースの名前として使います。

```
/memory                    # agentcore.json の名前を使う。なければ質問する
/memory UserContext        # 名前を指定して、特定の Memory リソースを対象にする
```

## 手順

### ステップ 1: プロジェクトを読み取る

`agentcore/agentcore.json` を読みます。次のものを探します。

- `memories` 配列 — Memory はすでに設定されているか
- `runtimes` 配列 — プロジェクトにどのエージェントがあり、どのフレームワークを使っているか
- プロジェクトの `name` — 環境変数名の組み立てに必要

**`agentcore/agentcore.json` が存在しない場合**は、近くに AgentCore のプロジェクト構造がないか（`agentcore/` ディレクトリを探す）確認します。見つからなければ、開発者の質問内容に基づいて、できるだけ役立つ回答を進めてください。コンテキストが足りないことで作業を止めないでください。質問が戦略の選択やコードパターンに関するものなら、直接回答します。「どの状況ですか?」と尋ねるのは、答えが本当にそれに依存する場合（たとえば、セットアップの種類によって CLI コマンドが異なる場合）だけにします。

### ステップ 2: 状況を判断する

**ケース A — Memory がまだ設定されていない**
`memories` 配列が空、または存在しません。ステップ 3（戦略の選択）に進みます。

**ケース B — Memory は設定済みで、組み込みが必要**
Memory は `agentcore.json` にあるものの、エージェントのコードがまだそれを使っていません。ステップ 5（組み込みコードの生成）に進みます。

**ケース C — Memory は設定済みで組み込み済み、想起のデバッグ**
次のように質問します。「何が起きていますか? エージェントに何を記憶してほしかったのか、実際には何をしたのかを教えてください。」
そのうえで、以下のデバッグのセクションにあるパターンを使って診断します。

**ケース D — プロジェクトなしで Memory について質問されている**
質問に直接答えます。戦略に関する質問なら、選択肢を説明します。コードに関する質問なら、実際の Memory ID に置き換える必要があるという注記を添えて、パターンを示します。

### ステップ 3: 戦略を選ぶ

選択肢を示し、どれがユースケースに合うかを開発者に尋ねます。これを省略しないでください。戦略を間違えると、費用が無駄になり、結果も悪くなります。

```
どの Memory 戦略がユースケースに合っていますか?

SEMANTIC
  適している用途: セッションをまたいでユーザーに関する事実を記憶する
  仕組み: 事実を抽出して埋め込みとして保存し、セッション開始時に
  類似度検索で関連するコンテキストを取得する
  コスト: 高め（セッションごとに埋め込みモデル + ベクトル検索）
  例: 「Alex は箇条書きを好み、フィンテック業界で働いていることを覚えておく」

USER_PREFERENCE
  適している用途: 明示的な設定や好みを記憶する
  仕組み: 構造化された好みのデータを抽出する。キーと値の取得に最適化されている
  コスト: 低め（構造化抽出のみで、ベクトル検索なし）
  例: 「私が好む回答の形式と言語を覚えておく」

SUMMARIZATION
  適している用途: 前回話した内容を記憶する
  仕組み: 会話履歴を要約に圧縮し、新しいセッションの開始時にその要約を挿入する
  コスト: 中程度（セッション終了時に要約モデルが実行される）
  例: 「前回の続きから始める」

EPISODIC
  適している用途: 時間にわたる出来事ややり取りの流れを記憶する
  仕組み: 時間的なコンテキストを付けて、やり取りのエピソード記録を保存する
  コスト: 中程度

よくある組み合わせ:
  SEMANTIC + USER_PREFERENCE  →  事実 + 好み（最も一般的）
  SEMANTIC + SUMMARIZATION    →  完全なエピソード記憶（最も高機能、最も高コスト）
  USER_PREFERENCE のみ         →  軽量な好みのストア

どの戦略（または組み合わせ）にしますか?
```

### ステップ 4: agentcore.json に Memory を追加する

CLI コマンドを実行して、プロジェクトの設定に Memory を追加します。

```bash
agentcore add memory --name <MemoryName> --strategies <STRATEGY1,STRATEGY2> --expiry 30
```

これにより `agentcore/agentcore.json` が更新されます。Memory リソースは、次に `agentcore deploy` を実行したときにプロビジョニングされ、有効になるまで 2〜5 分かかります。

生成される設定のエントリは、次のようになります。

```json
{
  "memories": [{
    "type": "AgentCoreMemory",
    "name": "MyMemory",
    "eventExpiryDuration": 30,
    "strategies": [
      {"type": "SEMANTIC"},
      {"type": "USER_PREFERENCE"}
    ]
  }]
}
```

**Memory 名のルール:** 英数字とアンダースコア、最大 48 文字、先頭は英字。

**デプロイ時に注入される環境変数:** `MEMORY_<UPPERCASENAME>_ID`
例: `UserContext` という名前の Memory → 環境変数 `MEMORY_USERCONTEXT_ID`

### ステップ 5: 組み込みコードを生成する

`app/<AgentName>/main.py`（または同等のエントリーポイント）を読んで、フレームワークを判別します。フレームワークごとに専用の統合パターンがあるので、一致するものを選んでください。

| フレームワーク | 推奨する統合方法 | 提供元 |
|---|---|---|
| Strands | `AgentCoreMemorySessionManager`（CLI テンプレート） | `bedrock_agentcore.memory.integrations.strands.*` |
| LangGraph | `AgentCoreMemorySaver` + `AgentCoreMemoryStore` | `langgraph-checkpoint-aws`（AWS が公式にメンテナンス） |
| OpenAI Agents SDK | `@function_tool` 経由の `MemoryClient` | `bedrock_agentcore.memory.MemoryClient` |
| Google ADK / Claude Agent SDK | 自前で実装 — `MemoryClient` を直接使う | 出荷前にエンドツーエンドで検証する |

> [!WARNING]
> Memory を初期化する前に、必ず MEMORY_ID 環境変数を確認してください。Memory は
> `agentcore dev` の間は利用できません — この環境変数は、デプロイ後にのみ設定されます。
> Memory が常に利用できると仮定したコードは、ローカル開発で通知なしに失敗します。

#### Strands — Session Manager パターン（新規プロジェクトに推奨）

```python
import os
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig, RetrievalConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager
from strands import Agent
from model.load import load_model  # `agentcore create` が生成します

app = BedrockAgentCoreApp()

# AgentCore はデプロイ後にこの環境変数を注入する。
# 形式: MEMORY_<UPPERCASENAME>_ID
MEMORY_ID = os.getenv("MEMORY_<UPPERCASENAME>_ID")
REGION = os.getenv("AWS_REGION", "us-east-1")

@app.entrypoint
def invoke(payload, context):
    actor_id = payload.get("userId", "default-user")
    session_id = getattr(context, "session_id", "default-session")

    session_manager = None
    if MEMORY_ID:
        # RetrievalConfig のパラメーター:
        #   top_k: 名前空間ごとに取得する Memory レコードの最大数（SDK のデフォルト: 10）
        #   relevance_score: 類似度のしきい値。0 = 何でも返す、1 = 完全一致（SDK のデフォルト: 0.2）
        # CLI テンプレートは、再現率よりも精度を優先するため、SDK のデフォルトとは異なる値を使っている:
        #   top_k=3 でコンテキストウィンドウの使用量を抑え、relevance_score=0.5 で低品質な一致を除外する。
        # 関連する事実を取りこぼす場合は下げ、無関係なものが出てくる場合は上げて調整する。
        memory_config = AgentCoreMemoryConfig(
            memory_id=MEMORY_ID,
            session_id=session_id,
            actor_id=actor_id,
            retrieval_config={
                f"/users/{actor_id}/facts": RetrievalConfig(top_k=3, relevance_score=0.5),
                f"/users/{actor_id}/preferences": RetrievalConfig(top_k=3, relevance_score=0.5),
            }
        )
        session_manager = AgentCoreMemorySessionManager(memory_config, REGION)

    agent = Agent(
        model=load_model(),
        session_manager=session_manager,  # None でも安全 — エージェントは Memory なしで動作する
        system_prompt="You are a helpful assistant.",
    )

    result = agent(payload.get("prompt", ""))
    return {"response": str(result)}

if __name__ == "__main__":
    app.run()
```

#### Strands — Hook パターン（既存のエージェントに Memory を追加する場合）

```python
import os
from bedrock_agentcore.memory import MemoryClient
from strands.hooks import AgentInitializedEvent, HookProvider, MessageAddedEvent

MEMORY_ID = os.getenv("MEMORY_<UPPERCASENAME>_ID")
memory_client = MemoryClient(region_name=os.getenv("AWS_REGION", "us-east-1")) if MEMORY_ID else None

class MemoryHook(HookProvider):
    def on_agent_initialized(self, event):
        """直近の会話のターンを、エージェントのコンテキストに読み込む。"""
        if not MEMORY_ID:
            return
        session_id = event.agent.state.get("session_id", "default")
        turns = memory_client.get_last_k_turns(
            memory_id=MEMORY_ID,
            actor_id="user",
            session_id=session_id,
            k=3
        )
        if turns:
            context = "\n".join([
                f"{m['role']}: {m['content']['text']}"
                for t in turns for m in t
            ])
            event.agent.system_prompt += f"\n\nPrevious conversation:\n{context}"

    def on_message_added(self, event):
        """各メッセージが処理されたあとに、Memory へ保存する。"""
        if not MEMORY_ID:
            return
        session_id = event.agent.state.get("session_id", "default")
        msg = event.agent.messages[-1]
        memory_client.create_event(
            memory_id=MEMORY_ID,
            actor_id="user",
            session_id=session_id,
            messages=[(str(msg["content"]), msg["role"])]
        )

    def register_hooks(self, registry):
        registry.add_callback(AgentInitializedEvent, self.on_agent_initialized)
        registry.add_callback(MessageAddedEvent, self.on_message_added)

# 既存のエージェントに追加する:
agent = Agent(
    # ... 既存の設定 ...
    hooks=[MemoryHook()] if MEMORY_ID else [],
    state={"session_id": "default"},
)
```

#### LangGraph — `langgraph-checkpoint-aws`（推奨）

LangGraph には、[`langgraph-checkpoint-aws`](https://pypi.org/project/langgraph-checkpoint-aws/) パッケージによる、**AWS が公式にメンテナンスする統合**があります。LangGraph の Memory モデルにうまく対応する、2 つの統合を提供します。

- **`AgentCoreMemorySaver`** — LangGraph のチェックポイントオブジェクト（会話の状態、実行グラフ、メタデータ）を AgentCore Memory に永続化します。これは LangGraph の短期/セッションの Memory です。
- **`AgentCoreMemoryStore`** — AgentCore の長期抽出（事実、好み、要約）のために、会話のメッセージを保存し、将来のセッションでエージェントがそれらの Memory を検索できるようにします。

グラフに `MemoryClient` の呼び出しを手動で組み込む代わりに、これらを使用してください。プロトコルの変換、アクター/セッションの対応付け、リトライのロジックを処理してくれます。

**インストール:**

```bash
pip install langgraph-checkpoint-aws
```

エージェントの実行ロールに必要な **IAM 権限**:

- `bedrock-agentcore:CreateEvent`
- `bedrock-agentcore:ListEvents`
- `bedrock-agentcore:RetrieveMemories`

**基本パターン — 短期のチェックポイントのみ:**

```python
import os
from langgraph.prebuilt import create_react_agent
from langgraph_checkpoint_aws import AgentCoreMemorySaver
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from model.load import load_model  # `agentcore create` が生成します

app = BedrockAgentCoreApp()

MEMORY_ID = os.getenv("MEMORY_<UPPERCASENAME>_ID")
REGION = os.getenv("AWS_REGION", "us-east-1")

# Memory が利用できる場合（デプロイ済み）にのみ、チェックポイントを組み込む
checkpointer = AgentCoreMemorySaver(MEMORY_ID, region_name=REGION) if MEMORY_ID else None

@app.entrypoint
async def invoke(payload, context):
    actor_id = payload.get("userId", "default-user")
    session_id = getattr(context, "session_id", "default-session")

    graph = create_react_agent(
        model=load_model(),
        tools=tools,
        checkpointer=checkpointer,  # None でも安全 — グラフは永続化なしで動作する
    )

    # LangGraph の RunnableConfig では、thread_id が AgentCore の session_id に、
    # actor_id が AgentCore の actor_id に、内部で対応付けられる
    config = {
        "configurable": {
            "thread_id": session_id,
            "actor_id": actor_id,
        }
    }

    result = await graph.ainvoke(
        {"messages": [("human", payload["prompt"])]},
        config=config,
    )
    return {"response": result["messages"][-1].content}
```

**完全なパターン — 短期 + 長期の取得:**

長期の Memory（AgentCore が抽出する事実、好み、要約）を使う場合は、`AgentCoreMemoryStore` を追加し、抽出用にメッセージを保存して、（必要に応じて）関連する Memory を取得する pre-model hook を組み合わせます。

```python
import os
import uuid
from langchain_core.messages import HumanMessage
from langchain_core.runnables import RunnableConfig
from langgraph.prebuilt import create_react_agent
from langgraph.store.base import BaseStore
from langgraph_checkpoint_aws import AgentCoreMemorySaver, AgentCoreMemoryStore

MEMORY_ID = os.getenv("MEMORY_<UPPERCASENAME>_ID")
REGION = os.getenv("AWS_REGION", "us-east-1")

checkpointer = AgentCoreMemorySaver(MEMORY_ID, region_name=REGION) if MEMORY_ID else None
store = AgentCoreMemoryStore(MEMORY_ID, region_name=REGION) if MEMORY_ID else None

def pre_model_hook(state, config: RunnableConfig, *, store: BaseStore):
    """最新の人間のメッセージを非同期抽出用に保存する。必要に応じて、好みを取得する。"""
    actor_id = config["configurable"]["actor_id"]
    thread_id = config["configurable"]["thread_id"]
    namespace = (actor_id, thread_id)

    messages = state.get("messages", [])
    for msg in reversed(messages):
        if isinstance(msg, HumanMessage):
            store.put(namespace, str(uuid.uuid4()), {"message": msg})
            break

    # 任意: ユーザーの好みを取得して、コンテキストに挿入する
    # preferences_ns = ("preferences", actor_id)
    # preferences = store.search(preferences_ns, query=msg.content, limit=5)

    return {"llm_input_messages": messages}

graph = create_react_agent(
    model=load_model(),
    tools=tools,
    checkpointer=checkpointer,
    store=store,
    pre_model_hook=pre_model_hook if store else None,
)
```

**config を指定して呼び出す:**

```python
config = {"configurable": {"thread_id": "session-1", "actor_id": "user-alice"}}
response = graph.invoke({"messages": [("human", "I prefer short answers.")]}, config=config)

# 同じアクターの新しいセッション — 長期の Memory が取得される
new_config = {"configurable": {"thread_id": "session-2", "actor_id": "user-alice"}}
response = graph.invoke({"messages": [("human", "Summarize my latest report.")]}, config=new_config)
```

AgentCore Memory が「I prefer short answers.」をユーザーの好みとして抽出するため、エージェントはセッションをまたいでそれを記憶します。完全なリファレンスは、[LangGraph 統合に関する AgentCore のドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-integrate-lang.html)を参照してください。

**低レベルの制御が必要な場合**（カスタムの取得クエリ、イベントの直接管理）は、`MemoryClient` にフォールバックします。

```python
from bedrock_agentcore.memory import MemoryClient

client = MemoryClient(region_name=REGION)
# client.create_event(...), client.retrieve_memories(...), client.get_last_k_turns(...)
```

`MemoryClient` を直接使うのは、チェックポイントやストアの抽象化がユースケースに合わない場合だけにしてください。

#### OpenAI Agents SDK — 関数ツールとしての Memory

OpenAI Agents SDK のパターンは、Memory を `@function_tool` で修飾した関数として公開することです。いつ読み書きするかは、エージェントが判断します。

```python
import os
from agents import Agent, Runner, function_tool
from bedrock_agentcore.memory import MemoryClient

MEMORY_ID = os.getenv("MEMORY_<UPPERCASENAME>_ID")
REGION = os.getenv("AWS_REGION", "us-east-1")
_client = MemoryClient(region_name=REGION) if MEMORY_ID else None

def _build_memory_tools(actor_id: str, session_id: str):
    """ファクトリー — アクターとセッションをツールのクロージャーに束縛する。"""

    @function_tool
    def recall_context(query: str, top_k: int = 3) -> str:
        """ユーザーに関する事実や好みを、長期 Memory から検索する。"""
        if not _client or not MEMORY_ID:
            return "Memory unavailable."
        try:
            memories = _client.retrieve_memories(
                memory_id=MEMORY_ID,
                namespace=f"/users/{actor_id}/facts",
                query=query,
                top_k=top_k,
            )
            return "\n".join(m.get("content", {}).get("text", "") for m in memories) or "No relevant memories."
        except Exception as e:
            return f"Memory error: {e}"

    @function_tool
    def save_fact(content: str) -> str:
        """事実を長期 Memory に保存する。"""
        if not _client or not MEMORY_ID:
            return "Memory unavailable."
        try:
            _client.create_event(
                memory_id=MEMORY_ID,
                actor_id=actor_id,
                session_id=session_id,
                messages=[(content, "ASSISTANT")],
            )
            return "Saved."
        except Exception as e:
            return f"Error: {e}"

    return [recall_context, save_fact]

@app.entrypoint
async def invoke(payload, context):
    actor_id = payload.get("userId", "default-user")
    session_id = getattr(context, "session_id", "default-session")

    agent = Agent(
        name="Assistant",
        instructions="Use recall_context at the start of each session to check what you know about the user. Use save_fact when the user tells you something worth remembering.",
        tools=_build_memory_tools(actor_id, session_id),
    )
    result = await Runner.run(agent, payload["prompt"])
    return {"response": result.final_output}
```

#### Google ADK と Claude Agent SDK — Memory 統合は自前で用意する

AgentCore Memory には、ADK や Claude Agent SDK 向けの、フレームワーク専用の統合はまだありません。また、サンプルのリポジトリにも、参照できる組み合わせパターンがありません。汎用の `MemoryClient` API を使い、フレームワークが持つ既存の拡張ポイントに組み込んでください。

- **Google ADK:** Memory の操作を ADK のツール（`Agent(tools=[...])` に渡す関数）として公開します。いつ呼び出すかは、ADK のエージェントが判断します。
- **Claude Agent SDK:** `query()` を、呼び出し前の Memory 読み込みと、呼び出し後の Memory 保存でラップします。取得したコンテキストを挿入する場所は、SDK の `ClaudeAgentOptions.system_prompt` です。

どちらのフレームワークでも、上の OpenAI Agents のパターンで示した `MemoryClient` API に従ってください。クライアントの呼び出し（`retrieve_memories`、`create_event`、`get_last_k_turns`）は同一です。フレームワークごとに異なるのは、それらを呼び出す場所だけです。

ADK や Claude SDK の Memory 統合を出荷する前に、デプロイ済みのエージェントでエンドツーエンドの流れを検証してください。

1. Memory を有効にしてデプロイする
2. 記憶させたい事実を伝えて、エージェントを呼び出す
3. 新しいセッションを開始する
4. もう一度呼び出し、エージェントがそれらの事実を想起することを確認する
5. `agentcore logs --runtime <AgentName> --query "memory" --since 1h --level error` で、Memory のエラーがないか確認する

動作するパターンを作れたら、次の開発者が一から調べなくて済むように、[`awslabs/agentcore-samples`](https://github.com/awslabs/agentcore-samples) への貢献を検討してください。

### ステップ 6: ローカル開発での制約と次のステップを説明する

必ず、次の注記を含めてください。

```
⚠️  Memory はローカル開発（agentcore dev）では利用できません。

環境変数 MEMORY_<NAME>_ID は、デプロイ後にのみ注入されます。上のコードは
これを適切に処理しており、環境変数が設定されていない場合は Memory なしで
動作します。

Memory をテストするには:
  agentcore deploy -y
  agentcore invoke "My name is Alex and I prefer concise answers"
  agentcore invoke "What do you know about me?"

長期 Memory（SEMANTIC または USER_PREFERENCE）を使う場合は、1 回目と
2 回目の呼び出しの間に 5〜30 秒待ってください。抽出は、各セッションの
終了後に非同期で実行されます。

セッション ID に関する注意: セッション ID には UUID（v4）を使ってください。
プラットフォームの最小長の要件（33 文字）を満たし、`agentcore invoke` が
デフォルトで生成するものでもあります。短い、または連番のセッション ID
（たとえば "session-1"、"test"）では、長期 Memory の抽出が通知なしに
失敗することがあります。
```

**開発者が SDK を直接使っている場合（CLI プロジェクトなし）**は、先に Memory リソースを作成する必要があります。

```python
from bedrock_agentcore.memory import MemoryClient

client = MemoryClient(region_name="us-east-1")

# Memory を作成し、ACTIVE になるまで待つ（2〜5 分かかる）
memory = client.create_memory_and_wait(
    name="UserMemory",
    strategies=[
        {"userPreferenceMemoryStrategy": {
            "name": "prefs",
            "namespaces": ["/user/preferences/"]
        }},
        {"semanticMemoryStrategy": {
            "name": "facts",
            "namespaces": ["/user/facts/"]
        }}
    ],
    event_expiry_days=30
)

MEMORY_ID = memory["id"]
print(f"Memory created: {MEMORY_ID}")
# これを環境変数として設定するか、テスト用にハードコードする:
# export MEMORY_ID=<value>
```

その後は、ステップ 5 と同じ組み込みコードを使い、`MEMORY_ID` を環境変数から読み取ります。

## Memory の想起をデバッグする

Memory が動いていたのに止まった場合や、一度も動かなかった場合:

**Memory を設定しているのに、エージェントが物事を忘れ続ける:**
最も多い原因は、Memory リソースは設定されているが、コードがセッション開始時にそこから読み込んでいないことです。エントリーポイントが、エージェントを作成した後ではなく、作成する前に `get_last_k_turns` を呼び出している（またはセッションマネージャーを使っている）か確認してください。また、環境変数 `MEMORY_<NAME>_ID` が設定されていることも確認してください。これはデプロイ後にのみ注入され、`agentcore dev` の間は設定されません。

**Memory がセッションをまたいで永続化されない:**

1. LTM の戦略（SEMANTIC、USER_PREFERENCE）が設定されているか確認する — SUMMARIZATION だけでは不十分
2. セッションが終了してから 5〜30 秒待って、新しいセッションを開始する — 抽出は非同期
3. Memory リソースが ACTIVE であることを確認する: `agentcore status --type memory`
4. セッション ID には UUID（v4）を使う — プラットフォームは最小 33 文字を要求する。"session-1" や "test" のような短い ID では、LTM が通知なしに失敗する。`agentcore invoke` は、要件を満たす ID をデフォルトで生成する。

**セッション開始時に Memory が読み込まれない:**

1. 環境変数 `MEMORY_<NAME>_ID` が設定されていることを確認する: `agentcore status --type memory --json`
2. actor_id がセッション間で一貫しているか確認する — Memory はアクターごとにスコープされる
3. retrieval_config の名前空間のパスが、書き込み時に使った名前空間と一致していることを確認する — 取得の名前空間は、戦略が抽出先とする名前空間と完全に一致していなければならない
4. CLI のデフォルトでは、末尾にスラッシュのないパスを使う（例: `/users/{actorId}/facts`）。Memory リソースを作成するときに名前空間のテンプレートをカスタマイズした場合は、選んだパターンをそのまま使う — 重要なのは、書き込み側と読み取り側の一貫性である。

**Memory のプロビジョニングが遅い:**
Memory は、`agentcore deploy` の後、ACTIVE になるまで 2〜5 分かかります。状態を確認します。

```bash
agentcore status --type memory
```

## S3 の配信/エクスポート用バケットは同じアカウントにある必要がある

Memory のエクスポート、セッションのトランスクリプト、Browser の録画出力のために S3 配信を設定する場合、出力先のバケットは、AgentCore のリソースと**同じ AWS アカウント**にある必要があります。サービスプリンシパルへのアクセスを許可する正しいバケットポリシーがあっても、クロスアカウントの S3 バケットは配信先としてサポートされません。

クロスアカウントのバケットを使おうとした場合の症状: IAM とバケットポリシーがクロスアカウントのアクセス用に正しく設定されていても、`CreateMemory`（または該当するリソースの作成呼び出し）が `ValidationException: Role does not have access to required S3 buckets` で失敗します。

**回避策:** AgentCore のリソースが書き込む先として、同じアカウントにバケットを作成します。別のアカウントでデータが必要な場合は、S3 レプリケーションや定期的なコピージョブで、同じアカウントのバケットからレプリケートします。

## エージェント間で Memory を共有する

Memory は最上位のリソースであり、単一のエージェントの配下にはありません。共有するには:

1. Memory リソースを 1 つ作成します: `agentcore add memory --name SharedMemory --strategies SEMANTIC`
2. 各エージェントのコードで、同じ環境変数を読みます: `MEMORY_SHAREDMEMORY_ID`
3. エージェント間で一貫した `actor_id` の方式を使います（たとえば、エンドユーザーの ID）

## クロスリージョン推論（データレジデンシー）

Memory のコンソリデーション（長期戦略のための抽出と要約）は、デフォルトでクロスリージョン推論を使用します。Memory の**データはプライマリリージョンに留まります**が、事実を抽出したりセッションを要約したりする**推論呼び出し**は、同じ地域内の別の AWS リージョンで実行される場合があります（たとえば、`us-east-1` → `us-east-2` または `us-west-2`。EU は EU 内に留まります）。

これが問題になるのは、次のような場合です。

- 保存場所を超えたデータレジデンシーの要件がある — 規制によっては、結果がどこに保存されるかだけでなく、推論をどこで実行できるかを制限するものがあります
- 契約で処理を単一のリージョンに限定している顧客向けに構築している
- 監査証跡で、各プロンプトをどのリージョンが処理したかを示す必要がある

**クロスリージョン推論に追加費用はなく、CloudWatch/CloudTrail のログには推論を行ったリージョンが含まれません。** `Memory`、`Policy`、`Evaluations` の各サービスでは、これがデフォルトの動作です。

**Memory でこれをオプトアウトするには:** **built-in-with-overrides** 戦略（[`memory-custom-strategy`](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-custom-strategy.html) を参照）を使い、モデルを特定のリージョンに固定します。overrides 戦略では、抽出とコンソリデーションに使うモデル ID を正確に指定できるため、リージョンを制御できます。

サポートされる地域と推論リージョンの対応は、AgentCore の展開に伴って変わります。ここに固定して書くのではなく、現在の一覧は[クロスリージョン推論のドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/cross-region-inference.html)で確認してください。

## CLI の範囲を超えて: API が必要な Memory の機能

CLI の `agentcore add memory` と `agentcore.json` がカバーするのは、戦略の選択、有効期限、基本的な設定です。一部の Memory の機能は API/SDK 専用であり、CLI からは利用できません。開発者がそのような機能を必要とする場合の進め方は、通常どおり CLI で Memory を作成してデプロイし、その後で追加の設定を boto3 や AWS CLI で適用する、というものです。

**リソースベースのポリシー**（クロスアカウントのアクセス、プリンシパル単位の制限）:

```python
import boto3, json

client = boto3.client("bedrock-agentcore-control")
memory_id = "<MEMORY_ID>"  # 取得方法: agentcore status --type memory --json

client.put_memory_resource_policy(
    memoryId=memory_id,
    policy=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"AWS": "arn:aws:iam::111122223333:root"},
            "Action": [
                "bedrock-agentcore:CreateEvent",
                "bedrock-agentcore:RetrieveMemories",
                "bedrock-agentcore:ListEvents"
            ],
            "Resource": "*"
        }]
    })
)
```

**カスタムの抽出モデル**（LTM の抽出に使うモデルを固定する — たとえば、データレジデンシーのため）:

`UpdateMemory` を通じて、"built-in with overrides" の戦略タイプを使います。完全な設定の形式は、[カスタム Memory 戦略のドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-custom-strategy.html)を参照してください。

**セルフマネージド戦略**（独自の抽出ロジックを持ち込む）:

これも API 専用です。`selfManagedMemoryStrategy` の設定については、AgentCore Memory のドキュメントを参照してください。

**ここで扱っていない Memory の機能に出会った場合**は、`awsknowledge` MCP サーバーが利用できるなら、それを使ってください。特定の API オペレーション（例: "AgentCore PutMemoryResourcePolicy"）を検索して、現在のパラメーターの形式を取得します。API の仕様は、リリースごとに進化します。

**一般的なルール:** `agentcore.json` に対応するフィールドがあるなら、CLI を使います。ないなら、CLI でリソースを作成してデプロイし、その後で追加の設定を boto3 で適用します。CLI が想定していないことを、CLI で無理にやろうとしないでください。

## 出力

- Memory リソースを追加した、更新後の `agentcore/agentcore.json`（CLI コマンド経由）
- 検出したフレームワークに適した、`app/<AgentName>/main.py` 用の組み込みコード
- ローカル開発での制約と、デプロイ後のテスト方法の説明

## 品質基準

- 生成されたコードが、`MEMORY_ID` が None の場合（ローカル開発）でもクラッシュしない
- 環境変数名が、`agentcore.json` の Memory リソース名と一致している（大文字、アンダースコア）
- フレームワーク固有のパターンを使っている — LangGraph のプロジェクトに Strands の hook を生成しない
- LTM の抽出に遅延があることを伝えている
- LTM 戦略を使う場合、セッション ID のガイダンスで UUID（v4）を推奨している（最小 33 文字）
