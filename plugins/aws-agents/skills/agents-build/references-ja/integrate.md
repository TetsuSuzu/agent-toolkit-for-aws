# integrate

開発者が、デプロイ済みのエージェントをアプリケーションから呼び出せるよう支援します。

## 使うタイミング

- 開発者がデプロイ済みのエージェントを持っており、自分のアプリから呼び出したい
- 開発者がエージェントの URL と認証情報を必要としている
- 開発者がエージェントからのストリーミングレスポンスを処理したい
- 開発者が、複数回の呼び出しにまたがる会話セッションを管理する必要がある
- 開発者が、エージェントを利用するフロントエンド、バックエンドサービス、CLI を構築している
- 呼び出し元とエージェントが別々の AWS アカウントにある（クロスアカウント呼び出し）

次の場合は使わないでください:

- 外部 API を呼び出すツールをエージェントに与える → `agents-connect` を使います
- エージェントをデプロイする → `agents-deploy` を使います
- エージェントのレスポンスをデバッグする → `agents-debug` を使います
- 本番向けにエージェントのエンドポイントを保護する → `agents-harden` を使います（ただし、このスキルはクライアント側の認証コードを扱います）

## 入力

`$ARGUMENTS` には次のものを指定できます。

- 言語またはフレームワーク: "from React"、"in Python"、"Node.js backend"
- 認証の希望: "using IAM"、"with JWT"
- 空 — スキルがプロジェクトのコンテキストを検出し、それに合わせて案内します

## 手順

### ステップ 1: デプロイ状況を確認する

`agentcore/agentcore.json` を読んで、エージェント名を取得します。そのうえで、デプロイ済みかどうかを確認します。

```bash
agentcore status --type agent
```

**デプロイされていない場合:** 「アプリから呼び出す前に、エージェントをデプロイする必要があります。先に `agentcore deploy` を実行するか、`agents-deploy` スキルで手順を確認してください。」

エージェントがデプロイされるまで、先に進まないでください。

### ステップ 2: エージェントのエンドポイントを取得する

```bash
agentcore fetch access --name <AgentName> --type agent
```

これは次のものを返します。

- **Agent Runtime ARN** — SDK から呼び出すときに必要
- **エンドポイント URL** — HTTPS で直接呼び出すときに使用
- **認証設定** — どの認証方式が設定されているか

出力にある認証タイプを控えておいてください。クライアントアプリがどのように認証するかは、これで決まります。

### ステップ 3: 認証方式を判断する

`agentcore/agentcore.json` から、エージェントの `authorizerType` フィールドを読みます（ランタイムのエントリの最上位にあるフィールドです。JWT の詳細は、同じランタイムの別オブジェクト `authorizerConfiguration` にあります）。

| 認証タイプ | クライアントの認証方法 | 適した用途 |
|---|---|---|
| **None**（デフォルト） | リクエストへの IAM SigV4 署名 | AWS 認証情報を持つバックエンドサービス |
| **AWS_IAM** | リクエストへの IAM SigV4 署名 | バックエンドサービス、Lambda からエージェントへの呼び出し |
| **CUSTOM_JWT** | Authorization ヘッダーの Bearer トークン | ID プロバイダーを持つ Web/モバイルアプリ |

**オーソライザーが設定されていない場合:** エージェントはデフォルトで IAM 認証を使用します。呼び出し元の ID には `bedrock-agentcore:InvokeAgentRuntime` 権限が必要です。

**CUSTOM_JWT の場合:** クライアントは、設定された ID プロバイダーが発行した JWT を送信します。エージェントは、セットアップ時に設定されたディスカバリー URL、許可された audience、許可されたクライアントに照らして、それを検証します。

### ステップ 4: クライアントコードを生成する

開発者の言語の希望（`$ARGUMENTS` から、または質問して確認）に基づいて、適切なクライアントコードを生成します。

#### Python (boto3) — IAM 認証

```python
import boto3
import json
from botocore.exceptions import ClientError

client = boto3.client("bedrock-agentcore", region_name="<REGION>")

try:
    response = client.invoke_agent_runtime(
        agentRuntimeArn="<AGENT_RUNTIME_ARN>",
        qualifier="DEFAULT",  # または特定のバージョン番号
        payload=json.dumps({
            "prompt": "Hello, what can you do?"
        }).encode(),
        runtimeSessionId="session-123",  # マルチターンの会話では再利用する
    )

    # ストリーミングレスポンスを処理する — response["response"] は StreamingBody
    stream = response["response"]
    if hasattr(stream, "iter_lines"):
        for line in stream.iter_lines():
            if line:
                print(line.decode(), end="", flush=True)
    else:
        # SDK のバージョンによっては生のバイト列が返る — 一括で読み込む
        content = stream.read()
        print(content.decode() if isinstance(content, bytes) else content)

except ClientError as e:
    code = e.response["Error"]["Code"]
    if code == "AccessDeniedException":
        # bedrock-agentcore:InvokeAgentRuntime 権限がない
        raise RuntimeError("Caller lacks InvokeAgentRuntime permission") from e
    elif code == "ValidationException":
        # ARN の誤り、ペイロード形式の不正、セッション ID の不正
        raise RuntimeError(f"Invalid request: {e}") from e
    elif code == "ThrottlingException":
        # 指数バックオフでリトライする
        raise
    else:
        raise
```

#### Python (HTTPS) — JWT 認証

```python
import requests

AGENT_URL = "<ENDPOINT_URL>"
JWT_TOKEN = "<TOKEN_FROM_YOUR_IDP>"

response = requests.post(
    AGENT_URL,
    headers={
        "Authorization": f"Bearer {JWT_TOKEN}",
        "Content-Type": "application/json",
    },
    json={"prompt": "Hello, what can you do?"},
    stream=True,
)

for chunk in response.iter_content(chunk_size=None):
    print(chunk.decode(), end="", flush=True)
```

#### JavaScript/TypeScript (AWS SDK) — IAM 認証

```typescript
import {
  BedrockAgentCoreClient,
  InvokeAgentRuntimeCommand,
} from "@aws-sdk/client-bedrock-agentcore";

const client = new BedrockAgentCoreClient({ region: "<REGION>" });

const response = await client.send(
  new InvokeAgentRuntimeCommand({
    agentRuntimeArn: "<AGENT_RUNTIME_ARN>",
    qualifier: "DEFAULT",
    payload: new TextEncoder().encode(
      JSON.stringify({ prompt: "Hello, what can you do?" })
    ),
    runtimeSessionId: "session-123",
  })
);

// response.response はストリーミングボディ
const decoder = new TextDecoder();
for await (const chunk of response.response) {
  process.stdout.write(decoder.decode(chunk));
}
```

#### JavaScript/TypeScript (fetch) — JWT 認証

```typescript
const AGENT_URL = "<ENDPOINT_URL>";
const JWT_TOKEN = "<TOKEN_FROM_YOUR_IDP>";

const response = await fetch(AGENT_URL, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${JWT_TOKEN}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ prompt: "Hello, what can you do?" }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  process.stdout.write(decoder.decode(value));
}
```

### ステップ 5: セッション管理

セッションの仕組みを説明します。

- **`runtimeSessionId`** — 複数回の呼び出しで同じ値を渡すと、会話のコンテキストが維持されます
- ユーザーの会話ごとに、一意のセッション ID を生成します（例: UUID）
- セッションはサーバー側にあります — エージェントは、そのセッション ID の会話履歴を記憶しています
- セッション ID を省略すると、各呼び出しはステートレスになります（会話の記憶がありません）

```python
import uuid

# 新しい会話
session_id = str(uuid.uuid4())

# 最初のターン
invoke(session_id, "What's the weather in Seattle?")

# 同じ会話の続き
invoke(session_id, "What about tomorrow?")

# 新しい会話 — 新しいセッション
new_session_id = str(uuid.uuid4())
invoke(new_session_id, "Different topic entirely")
```

### ステップ 6: プロトコル固有のガイダンス

`agentcore/agentcore.json` から、エージェントの `protocol` を読みます。

**HTTP（デフォルト）の場合:** 上記のパターンがそのまま適用されます。

**MCP の場合:** エージェントは MCP エンドポイントを公開します。クライアントは MCP プロトコル（Streamable HTTP）で接続します。開発者に、使用する言語の MCP クライアントライブラリを案内してください。

**A2A の場合:** エージェントは、`/.well-known/agent-card.json` にカードを持つ Agent-to-Agent エンドポイントを公開します。呼び出し側のエージェントは、カードを通じて機能を発見し、JSON-RPC 2.0 で通信します。A2A の統合パターンは、このスキルの [`references/multi-agent.md`](multi-agent.md) を参照してください。

### ステップ 7: 正しく見えるのに失敗する統合パターン

サポートケースで頻繁に出てくる 2 つのパターンを、ここで明示しておきます。

**URL エンコードした Runtime ARN を使う API Gateway の `/{proxy+}`。** リソースを `/{proxy+}` とし、統合 URI をエンコードした Runtime ARN にした API Gateway REST API を AgentCore Runtime の前段に置くと、一見うまく動くように見えます。デプロイは成功し、短いリクエストは返ってきます。しかし、長いリクエストは約 2 分で失敗し、ログに `Integration closed connection prematurely` と出力されます。`integrationTimeoutInMillis` の設定は関係ありません。`HTTP_PROXY` は汎用の転送統合であり、SDK クライアントのように SigV4、ストリーミング、セッションのセマンティクスを処理しません。

代わりに、次のいずれかを使用してください。

- クライアントから `bedrock-agentcore` SDK で Runtime を直接呼び出します（上記のステップ 4）。これが想定された方法です。
- レート制限、公開 HTTPS エンドポイントなどの理由で API Gateway が必要な場合は、API Gateway と Runtime の間に Lambda を置きます。Lambda がリクエストを受け取り、`invoke_agent_runtime` を呼び出して、レスポンスをストリーミングで返します。Lambda の実行ロールには `bedrock-agentcore:InvokeAgentRuntime` が必要です。API Gateway には同期レスポンスに 29 秒というハードな上限があるため、この構成は処理が速いエージェントでのみ機能します。複数のステップを踏むものには、SDK を直接使う方法を選んでください。

**同期的なエージェントレスポンスのために Lambda を前段に置くと、短いタイムアウトの上限に当たります。** `Client → API Gateway → Lambda → Runtime` という連鎖は、API Gateway の同期レスポンスの制限により、約 29 秒が上限になります。推論を行うエージェント、複数のツールを呼び出すエージェント、重いモデルを使うエージェントは、いずれもこれを超えます。Runtime をラップする Lambda でタイムアウトが発生している場合、通常は Lambda をやめて、クライアントから Runtime を直接呼び出すのが解決策です。Runtime はストリーミングレスポンスをネイティブにサポートしており、そもそもチームが Lambda を追加する理由は、たいていこれです。

### ステップ 8: クロスアカウント呼び出し

呼び出し元とは別の AWS アカウントにあるエージェントの呼び出しには、標準的な AWS のクロスアカウント IAM パターンを使います。AgentCore 固有の仕組みは不要です。呼び出し元のアカウントが、エージェントのアカウントにあるロールを引き受け、一時的な認証情報を取得して、呼び出しリクエストへの署名に使用します。

**エージェントのアカウント（アカウント B）での設定:**

呼び出し元のアカウントを信頼し、ランタイムを呼び出す権限を持つ IAM ロールを作成します。

```json
// 信頼ポリシー — このロールを引き受けられる主体
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<CALLER_ACCOUNT_ID>:root"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {"sts:ExternalId": "<unique-external-id>"}
    }
  }]
}
```

```json
// 権限ポリシー — このロールが実行できること
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "bedrock-agentcore:InvokeAgentRuntime",
    "Resource": "arn:aws:bedrock-agentcore:<REGION>:<AGENT_ACCOUNT_ID>:runtime/<RUNTIME_NAME>-*"
  }]
}
```

信頼ポリシーの `Principal` は、できるだけ狭く指定してください（概念実証を超える用途では、`:root` ではなく、呼び出し元アカウント内の特定のロール ARN にします）。混乱した代理（confused deputy）問題を防ぐために、`ExternalId` を使用してください。

**呼び出し元のアプリ（アカウント A）:**

```python
import boto3
import json

# アカウント B のロールを引き受ける
sts = boto3.client("sts")
assumed = sts.assume_role(
    RoleArn="arn:aws:iam::<AGENT_ACCOUNT_ID>:role/<ROLE_NAME>",
    RoleSessionName="agent-invoker",
    ExternalId="<unique-external-id>",
)
creds = assumed["Credentials"]

# 一時的な認証情報を使ってランタイムを呼び出す
agentcore = boto3.client(
    "bedrock-agentcore",
    region_name="<REGION>",
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
)

response = agentcore.invoke_agent_runtime(
    agentRuntimeArn="arn:aws:bedrock-agentcore:<REGION>:<AGENT_ACCOUNT_ID>:runtime/<RUNTIME_NAME>",
    qualifier="DEFAULT",
    payload=json.dumps({"prompt": "hello"}).encode(),
    runtimeSessionId="session-123",
)
```

**本番運用での注意点:**

- 引き受けたロールの認証情報はキャッシュしてください。認証情報はセッションの有効期間（デフォルトで 1 時間）の間は有効です。リクエストのたびではなく、期限が近づいたときに再度引き受けてください。
- 呼び出し元の環境が AWS の config プロファイルをサポートしている場合は、`role_arn` と `source_profile` を使ったプロファイルを指定した boto3 の `Session` で、これを自動化できます。コード内の `assume_role` は、明示的な方法です。
- 呼び出し元が Lambda、ECS タスク、EC2 インスタンスの場合、AssumeRole 権限を付与されるのは、その実行ロール/タスクロールです。そのロールが、アカウント B の信頼ポリシーに記載するものになります。
- ランタイム自身のリソースポリシー（ある場合）は、IAM とは別のものです。通常、クロスアカウントではリソースポリシーは不要です。アクセスを許可するのは、アカウント B の IAM ロールです。

## 稼働中のエージェントセッション内でシェルコマンドを実行する（`InvokeAgentRuntimeCommand`）

エージェントのセッションが稼働している間は、**同じセッションの microVM** の中でシェルコマンドを実行し（同じファイルシステム、同じ環境変数、同じネットワーク名前空間）、その出力をストリーミングで受け取れます。これは `InvokeAgentRuntime`（エージェントの推論ループを動かすもの）の代わりではなく、それと並んで使うものです。

これが役立つ場面:

- コーディング/DevOps 系のエージェントで、アプリ側が LLM に推論させる代わりに、決まった処理（git pull、ビルド、テスト、ファイルシステムの確認）を実行する場合
- エージェントを実行する前に、セッションのファイルシステムに初期データを入れる場合（データセットを `/tmp` に置いてから、エージェントを呼び出して分析させる）
- 止まっている、または挙動のおかしいセッションのデバッグ — エージェントを介さず外部から `ps`、`ls`、`cat /tmp/log` を実行する
- スクリプト化されたコマンドの信頼性と、ウォーム状態のセッションのコンテキストの両方が欲しいあらゆるワークフロー

適さない場面:

- ユーザーのために任意のコードを実行する新しいセッションを起動する — 代わりに組み込みツールの [`code-interpreter.md`](code-interpreter.md) を使ってください。専用に作られており、サンドボックスの仕組みも異なり、エージェントのセッションを消費しません
- 無関係な呼び出し元に許してはならない処理の実行 — コマンドは、ランタイムの実行ロールとファイルシステムで実行されます

**必要な IAM 権限:** ランタイム ARN に対する `bedrock-agentcore:InvokeAgentRuntimeCommand`。これは `InvokeAgentRuntime` とは**別の**アクションです。必要とする呼び出し元だけに、明示的に絞って付与してください。

```python
import boto3

client = boto3.client("bedrock-agentcore", region_name="<REGION>")

response = client.invoke_agent_runtime_command(
    agentRuntimeArn="<AGENT_RUNTIME_ARN>",
    qualifier="DEFAULT",
    runtimeSessionId="session-123",   # 既存のセッションでなければならない
    command="ls -la /tmp && cat /tmp/status.json",
)

# 出力は response["response"] 上で HTTP/2 によりストリーミングされる
for chunk in response["response"].iter_chunks():
    print(chunk.decode(), end="", flush=True)
```

**セッションが存在している必要があります。** `InvokeAgentRuntimeCommand` は、稼働中のセッションにアタッチします。セッションを新たに作ることはありません。セッションが期限切れだったり、そもそも存在しなかったりする場合、呼び出しは失敗します。先にエージェントを呼び出して（セッションを開始して）から、そのセッション ID を以降のコマンド呼び出しに使用してください。

**同じ microVM、同じファイルシステム。** コマンドが書き込んだファイルは、次のエージェント呼び出しで見えます。逆も同様です。これを使って成果物を事前に読み込み、エージェントにそれらを対象に推論させることができます。セッションの分離は引き続き適用されるため、他のセッションからはこれらのファイルは見えません。

> [!WARNING]
> InvokeAgentRuntimeCommand は、稼働中のエージェントセッション内で、ランタイムの実行ロールの
> 全権限を使って任意のシェルコマンドを実行します。シェルへのアクセスが明確に必要な場合を除き、
> bedrock-agentcore:InvokeAgentRuntime を持つ主体に
> bedrock-agentcore:InvokeAgentRuntimeCommand を付与しないでください。
> コマンド実行用には、必ず別の IAM ポリシーを作成してください。
> InvokeAgentRuntimeCommand の呼び出しについては、必ず CloudTrail のログを有効にしてください。
> コマンドをユーザーが入力した内容から組み立てる場合は、検証とサニタイズを行ってください。
> これはコマンドインジェクションの攻撃対象になります。

**IAM の分離:** `InvokeAgentRuntimeCommand` は、`InvokeAgentRuntime` とは別の IAM アクションです。シェルアクセスが必要な呼び出し元にだけ付与し、エージェントを呼び出せるすべての ID には付与しないでください。最小限の例:

```json
{
  "Effect": "Allow",
  "Action": "bedrock-agentcore:InvokeAgentRuntimeCommand",
  "Resource": "arn:aws:bedrock-agentcore:<REGION>:<YOUR_ACCOUNT_ID>:runtime/<RUNTIME_NAME>-*"
}
```

これは、`InvokeAgentRuntime` を許可するポリシーとは別の IAM ポリシーにしてください。エージェントのセッション内でコマンドを実行する必要があることが明確なロールにのみ、アタッチします。

**コマンドインジェクション:** 上のコード例は、意図的にハードコードしたコマンド文字列を使っています。実際の利用でユーザーが入力した内容からコマンドを組み立てる場合は、渡す前に検証してください。`&&`、`;`、`$(...)`、バッククォート、`|`、その他のシェルメタ文字を含む文字列は拒否します。サニタイズしていないユーザー入力を `InvokeAgentRuntimeCommand` に渡すことは、コードを直接実行される脆弱性になります。

**CloudTrail による監視:** 想定外の `InvokeAgentRuntimeCommand` 呼び出しを通知する EventBridge ルールを有効にします。

```bash
aws events put-rule \
  --name AgentCoreCommandExecution \
  --event-pattern '{"source":["aws.bedrock-agentcore"],"detail-type":["AWS API Call via CloudTrail"],"detail":{"eventName":["InvokeAgentRuntimeCommand"]}}' \
  --state ENABLED
```

この権限を持つ呼び出し元が侵害されると、エージェントのファイルシステムの読み書き、エージェントが到達できるあらゆるネットワークリソースへのアクセス、実行ロールの認証情報の利用が可能になります。CloudTrail のログは、最低限の検知の基準です。

## 参考となる統合例

よく使われる 2 つの統合先には、公開された再利用可能なパターンがあります。統合レイヤーを一から作る代わりに、そこから始められます。

**Slack。** [Integrating Amazon Bedrock AgentCore with Slack](https://aws.amazon.com/blogs/machine-learning/integrating-amazon-bedrock-agentcore-with-slack/) では、任意の AgentCore エージェントを Slack ワークスペースに組み込む、再利用可能な統合レイヤーを解説しています。アーキテクチャ（API Gateway → Lambda → SQS → AgentCore）は、非同期処理によって Slack の 3 秒の Webhook タイムアウトに対応します。1 つ目の Lambda が Slack の署名を検証して即座に応答を返し、2 つ目が「処理中...」というプレースホルダーを投稿し、3 つ目がエージェントを呼び出して、プレースホルダーを実際のレスポンスに置き換えます。このパターンでは、Slack のスレッドのタイムスタンプを AgentCore Memory のセッション ID に、Slack のユーザー ID をアクター ID に対応付けるため、同じスレッド内で会話のコンテキストが時間をまたいで保持されます。統合レイヤーはエージェントから切り離されているので、Slack 側のインフラに手を加えずに、任意のエージェント（FinOps、DevOps、インシデント対応など）に差し替えられます。デプロイは `cdk deploy` 1 回で完了します。

**Microsoft Teams。** 同じ非同期処理のアーキテクチャ（API Gateway → Lambda → キュー → AgentCore）が Teams にも適用できます。Teams 固有の設定（Bot Framework の登録、ボットチャネルの構成）については、[How Amazon Bedrock transforms Microsoft Teams conversations into actionable insights](https://aws.amazon.com/blogs/industries/how-amazon-bedrock-transforms-microsoft-teams-conversations-into-actionable-insights/) を参照してください。上記の Slack のパターンをすでに構築している場合、Teams 版は主に、Webhook の検証部分とレスポンスの整形部分が異なるだけです。

どちらのパターンも、「タイムアウトの短い Webhook プラットフォーム」という問題を同じ方法で解決します。チャットプラットフォームには即座に ack とプレースホルダーを返し、実際のエージェント呼び出しは非同期で行い、準備ができたらレスポンスでプレースホルダーを置き換えます。ここに挙げていない 3 つ目のチャットプラットフォームと統合する場合は、どちらかのブログをテンプレートとして使ってください。

## 出力

- エージェントのエンドポイント URL と ARN
- 認証方式の説明と、クライアント側のコード
- 開発者が希望する言語の、動作するクライアントコード
- セッション管理のガイダンス
- 該当する場合は、プロトコル固有の注意事項

## 品質基準

- クライアントコードが正しい SDK クライアントを使っている（`bedrock-agent` ではなく `bedrock-agentcore`）
- 認証方式が、エージェントに設定されているものと一致している
- ストリーミングレスポンスの処理を含んでいる（リクエスト/レスポンスだけではない）
- セッション ID のパターンを説明している
- コードが完結していて実行できる — インポートと基本的なエラー処理を含む
