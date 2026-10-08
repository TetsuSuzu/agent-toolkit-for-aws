---
name: agents-harden
description: >
  エージェントを本番向けに準備するときに使います — IAM のスコープ設定、インバウンド
  認証(JWT、SigV4)、シークレット管理、コールドスタートの最適化、セッションの
  ライフサイクル、レート制限、入力の検証、クォータに関するガイダンス。トリガー:
  "production checklist"、"harden agent"、"production ready"、"secure
  agent"、"inbound auth"、"going live"、"cold start optimization"、"session
  lifecycle"、"StopRuntimeSession"、"quota"、"throttling"、"maxVms"、
  "rate limit"、"security audit of outbound API calls"、"gateway target
  audit for production"、"restrict who can call"、"lock down endpoint"、
  "only our app can call"。
  Cedar によるツール制限のポリシーには使いません — agents-connect を使います。
  品質の測定には使いません — agents-optimize を使います。アウトバウンドの
  認証情報の保存や API キーの組み込みには使いません — agents-connect を使います。
  A2A のエージェント間認証には使いません — agents-build を使います。コールドスタートの
  観察と診断(最適化ではないもの)は、agents-debug に振り分けます。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# harden

AgentCore のエージェントを、本番向けに準備します — セキュリティ、信頼性、パフォーマンス。

## 使うタイミング

- エージェントを本番に移行しようとしている
- ローンチ前に確認すべきことのチェックリストが欲しい
- エージェントを呼び出せる主体を制限したい
- デフォルトから、IAM 権限を絞り込みたい
- スロットリングやクォータのエラーが発生している([`references-ja/limits.md`](references-ja/limits.md) を読み込む)
- ワークロードに合わせて、セッションのライフサイクルを調整する必要がある
- エージェント内で、長時間実行されるバックグラウンド処理を実行している

## 入力

引数は不要です。スキルはプロジェクトの設定を読み取り、そのプロジェクトに固有の所見を含むチェックリストを作成します。

## 手順

### ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。それより古い場合は、先に進む前に `agentcore update` を実行するよう、開発者に伝えてください。

### ステップ 1: プロジェクトを読み取る

`agentcore/agentcore.json` を読み、次のことを把握します。

- 設定されているリソース(Memory、Gateway、認証情報、評価器)
- 使われているフレームワーク
- 設定されているネットワークモード(PUBLIC または VPC)

### ステップ 2: チェックリストを一通り確認する

各カテゴリを順に確認し、プロジェクトに固有の所見を報告します。

---

## IAM: 権限を絞り込む

自動作成される実行ロールは、Bedrock への広範なアクセス(`arn:aws:bedrock:*::foundation-model/*`)を持っています。本番では、エージェントが使う特定のモデルに絞り込んでください。

**現在の実行ロールを確認する:**

```bash
agentcore status --json | jq -r '.runtimes[0].executionRoleArn'
```

**本番向けに推奨する Bedrock のポリシー:**

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": [
    "arn:aws:bedrock:<REGION>::foundation-model/anthropic.claude-sonnet-4-5-20250929-v1:0"
  ]
}
```

リソースの ARN は、エージェントが使う特定のモデルに置き換えてください。

**ECR アクセス:** 特定のリポジトリに絞り込みます。

```json
{
  "Effect": "Allow",
  "Action": ["ecr:BatchGetImage", "ecr:GetDownloadUrlForLayer"],
  "Resource": "arn:aws:ecr:<REGION>:<YOUR_ACCOUNT_ID>:repository/bedrock-agentcore-<AGENT_NAME>-*"
}
```

**信頼ポリシー:** 実行ロールの信頼ポリシーが、自分のアカウントに絞り込まれていることを確認します。

```json
{
  "Principal": {"Service": "bedrock-agentcore.amazonaws.com"},
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {"aws:SourceAccount": "<YOUR_ACCOUNT_ID>"},
    "ArnLike": {"aws:SourceArn": "arn:aws:bedrock-agentcore:<REGION>:<YOUR_ACCOUNT_ID>:*"}
  }
}
```

**ランタイムのリソースベースポリシー**(API 専用): IAM ロールや JWT 認証が提供する範囲を超えて、どの主体がランタイムを呼び出せるかを細かく制御するには、boto3 で `PutAgentRuntimeResourcePolicy` を使います。これは CLI や `agentcore.json` では扱えません。`awsknowledge` MCP サーバーが利用できる場合は、現在の API の形式を調べるのに使ってください。

---

## シェルアクセス: `InvokeAgentRuntimeCommand` を個別に絞り込む

プロジェクトで `InvokeAgentRuntimeCommand`([`agents-build/references-ja/integrate.md`](../agents-build/references-ja/integrate.md) を参照)を使う場合は、その IAM 権限を `InvokeAgentRuntime` とは別に監査してください。この 2 つのアクションは、影響の範囲が異なります。`InvokeAgentRuntimeCommand` は、稼働中の microVM 内でランタイムの実行ロールの全権限を使って任意のシェルを実行するものです。呼び出し元は、ファイルシステムの読み書き、エージェントが到達できるあらゆるネットワークリソースへのアクセス、実行ロールの認証情報へのアクセスが可能です。

**どの主体が権限を持っているか確認する:**

```bash
# アカウント内のカスタマー管理ポリシーを一覧表示し、それぞれで InvokeAgentRuntimeCommand を調べる
aws iam list-policies --scope Local \
  --query 'Policies[*].[PolicyName, Arn, DefaultVersionId]' \
  --output table
# その後、気になるポリシーごとに:
aws iam get-policy-version \
  --policy-arn <POLICY_ARN> \
  --version-id <VERSION_ID> \
  --query 'PolicyVersion.Document'
```

別の方法として、IAM コンソールを使います: **IAM → Policies → Filter by type: Customer managed** → ポリシーの JSON エディタで `InvokeAgentRuntimeCommand` を検索します。

**コマンドの呼び出し元には、別の IAM ポリシーを用意する** — `InvokeAgentRuntime` を許可するポリシーとは分けます。

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "bedrock-agentcore:InvokeAgentRuntimeCommand",
    "Resource": "arn:aws:bedrock-agentcore:<REGION>:<YOUR_ACCOUNT_ID>:runtime/<RUNTIME_NAME>-*"
  }]
}
```

**CloudTrail によるアラートを有効にする。** `InvokeAgentRuntimeCommand` が呼び出されたときに、セキュリティチームに通知する EventBridge ルールを作成します。

```bash
aws events put-rule \
  --name AgentCoreCommandExecution \
  --event-pattern '{"source":["aws.bedrock-agentcore"],"detail-type":["AWS API Call via CloudTrail"],"detail":{"eventName":["InvokeAgentRuntimeCommand"]}}' \
  --state ENABLED
```

**呼び出し側のコードのどこかで、ユーザーの入力からコマンドを組み立てている場合:** 渡す前に検証します — `&&`、`;`、`$(...)`、バッククォート、`|`、その他のシェルメタ文字を含む文字列は拒否してください。

---

## インバウンド認証: エージェントを呼び出せる主体を制御する

デフォルトでは、エージェントはインバウンド認証に AWS IAM(SigV4)を使います。本番では、これが正しく設定されていることを確認してください。

**現在の認証設定を確認する:**

```bash
agentcore status --runtime <AgentName> --json | jq '.runtimes[0].authorizerConfig'
```

**オプション:**

`AWS_IAM`(デフォルト)— 呼び出し元は、リクエストに SigV4 で署名する必要があります。内部サービスや AWS ネイティブのクライアントに適しています。

`CUSTOM_JWT` — 呼び出し元は、自分の ID プロバイダーが発行した JWT を提示します。Web/モバイルアプリや外部のクライアントに適しています。

```bash
agentcore add agent \
  --name MyAgent \
  --authorizer-type CUSTOM_JWT \
  --discovery-url https://your-idp.example.com/.well-known/openid-configuration \
  --allowed-audience my-api \
  --allowed-clients my-client-id
```

> [!WARNING]
> 本番で `--authorizer-type NONE` を使わないでください。エージェントへの認証なしの
> アクセスを許可し、エンドポイントの URL を知っている人なら誰でも呼び出せるようになります。
> 必ず AWS_IAM または CUSTOM_JWT を使ってください。本番で NONE を見つけた場合は、
> ただちに変更してください。

### `allowedClients` と `allowedAudience` の選択

これは、JWT の設定ミスで最も多いものです。正しい選択は、ID プロバイダーが発行するトークンの中身に依存します。

**サンプルのトークンをデコードして**(ID プロバイダー側で、または `jwt.io` で)ペイロードを確認します。

- トークンに `client_id` クレームがあり、`aud` クレームがない → ランタイムに **`allowedClients`** を設定する
- トークンに `aud` クレームがある → ランタイムに **`allowedAudience`** を設定する
- 両方ある → `allowedAudience` を使う。`aud` クレームは標準の OIDC の audience フィールドなので、それを主なチェックとして使います。

間違ったほうを選ぶと、有効なトークンでも呼び出しが 403 を返します。ランタイムは、トークンに存在しないクレームに対して検証しているためです。

### issuer と discovery URL のプレフィックスの要件

AgentCore は、OIDC ディスカバリーの仕様(RFC 8414 §3)を強制します。ディスカバリードキュメントの `issuer` の値は、ディスカバリーエンドポイントの URL のプレフィックスでなければなりません。

つまり、ディスカバリー URL が `https://qa.example.com/.well-known/openid-configuration` なら、そのドキュメントの `issuer` フィールドは `https://qa.example.com` で始まる必要があります。ドキュメントが `https://example.com`(サブドメインなし)のような issuer を示している場合は、検証に失敗します。

一部のエンタープライズ向けの ID プロバイダー(PingFederate、Paylocity、一部の Keycloak の構成)は、環境固有のサブドメインでディスカバリーエンドポイントをホストしながら、本番レベルの issuer を示します。このパターンは、RFC 8414 のプレフィックスのルールと互換性がありません。

対処の選択肢:

1. **ID プロバイダーのディスカバリーエンドポイントを、issuer に合わせる** — issuer と同じオリジンからディスカバリーを提供します。
2. **ランタイムを、実際のディスカバリー URL のドメインに向ける** — トークンの issuer と一致するサブドメインで、ランタイムのディスカバリー URL を設定します。

### JWT 認証の失敗をデバッグする

呼び出しが 403 で失敗する場合は、どのチェックが失敗しているのかを絞り込みます。

**`Authorization method mismatch`** — ランタイムの認証タイプと、リクエストの認証タイプが一致していません。2 つのケースがあります。

- ランタイムが `AWS_IAM`(またはオーソライザーなし)で設定されているのに、呼び出し元が Bearer トークンを送っている → ランタイムを `CUSTOM_JWT` で設定し直すか、呼び出し元に SigV4 を使わせます。
- ランタイムが `CUSTOM_JWT` で設定されているのに、呼び出し元のリクエストが SigV4 で署名されている → SDK や環境が、Bearer トークンと一緒に SigV4 のヘッダーを挿入している可能性が高いです。アウトバウンドのリクエストに `X-Amz-Date`、`X-Amz-Security-Token`、`Authorization: AWS4-HMAC-SHA256` がないか確認してください。SigV4 の経路を取り除き、Bearer トークンだけを送ります。

**`Invalid inbound token`**(または類似のもの)— トークンが JWT バリデーターに拒否されました。次の順に確認します。

1. **issuer と discovery URL のプレフィックス**(上記)— トークンの `iss` クレームが、ディスカバリー URL のオリジンと一致していることを確認する
2. **`allowedClients` と `allowedAudience`** — ランタイムが、トークンの形式に合った正しいクレームで設定されているか
3. **JWKS への到達性** — AgentCore は、ディスカバリードキュメントに記載された `jwks_uri` に到達できるか。公開で到達可能である必要がある
4. **トークンの期限切れ** — トークンをデコードして、`exp` を現在時刻と照らして確認する
5. **署名アルゴリズムのサポート** — ID プロバイダーによっては、普遍的にはサポートされていないアルゴリズム(PS256、ES384 など)で署名します。ID プロバイダーがサポートするアルゴリズムを確認し、互換性が問題なら RS256 に切り替える。

これらをすべて除外してから、サービス側の問題として扱ってください。

---

## エラー処理: 適切に失敗させる

エージェントのコードが、内部の詳細を露出せずにエラーを処理していることを確認します。

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload, context):
    try:
        # エージェントのロジック
        return {"response": result}
    except Exception as e:
        # 完全なエラーは内部にログとして記録する
        app.logger.error(f"Agent error: {e}", exc_info=True)
        # 呼び出し元には、安全なメッセージを返す
        return {"error": "An error occurred. Please try again."}

if __name__ == "__main__":
    app.run()
```

**確認すること:** エラーを黙って握りつぶす裸の `except` ブロック、スタックトレースや内部の詳細を呼び出し元に露出するエラーメッセージ、ツール呼び出しのコードでのエラー処理の欠落。

---

## 入力の検証とレート制限

エージェントのエントリーポイントは、呼び出し元から任意のペイロードを受け取ります。処理する前に、入力を検証してください。

```python
@app.entrypoint
def invoke(payload, context):
    prompt = payload.get("prompt", "")

    # 入力を検証する
    if not prompt or not isinstance(prompt, str):
        return {"error": "Missing or invalid 'prompt' field"}
    if len(prompt) > 10000:
        return {"error": "Prompt exceeds maximum length (10,000 characters)"}

    # サニタイズ — 制御文字や過剰な空白を取り除く
    prompt = " ".join(prompt.split())

    # 検証済みの入力で処理を進める
    result = agent(prompt)
    return {"response": str(result)}
```

**検証すること:**

- 必須フィールドが存在し、期待する型であること
- 文字列の入力が、妥当な長さの上限を超えていないこと(モデルへのトークン爆弾を防ぐ)
- 数値の入力が、期待する範囲内であること
- ユーザーが指定する ID(actor_id、session_id)が、期待する形式に一致していること

**レート制限:** AgentCore Runtime には、組み込みの呼び出しレート制限があります(デフォルトはエージェントあたり 25 TPS — [`references-ja/limits.md`](references-ja/limits.md) を参照)。アプリケーションレベルのレート制限(ユーザーごと、テナントごと)は、エージェントのコード自体ではなく、呼び出し側のアプリケーションや API Gateway の層で実装してください。エージェントは、リクエストが届く時点で、すでにレート制限されていると仮定するべきです。

---

## シークレット: コードに認証情報を入れない、ランタイムの環境変数にシークレットを入れない

確認すべき失敗のパターンが 2 つあります。

### 1. エージェントのコードにハードコードされたシークレット

```bash
# エージェントのコードから、よくあるシークレットのパターンを検索する
grep -r "sk-\|api_key\s*=\s*['\"]" app/ --include="*.py"
grep -r "password\s*=\s*['\"]" app/ --include="*.py"
```

### 2. ランタイムの環境変数から取得されるシークレット

AgentCore Runtime の環境変数は、Vault で保護されて**いません**。開発者が(CDK、boto3 の `UpdateAgentRuntime` などで)ランタイムの環境に入れたものは、シークレットではなく、平文の設定値です。次のパターンを監査します。

```bash
# 名前がシークレットを示唆する os.getenv / os.environ の呼び出しを検出する
grep -rE "os\.(getenv|environ).*(TOKEN|SECRET|KEY|PASSWORD|CREDENTIAL)" app/ --include="*.py"
```

プラットフォームが注入するシークレットではない識別子(例: `MEMORY_*_ID`、`AGENTCORE_GATEWAY_*_URL`、`AWS_REGION`、下流のエージェントの ARN)は問題なく、許可リストに一致させるべきではありません。ヒットしたものを確認し、シークレットがないことを確かめます。

**正しいパターン:** 各アウトバウンドの認証情報を `agentcore add credential` で登録し、コードでは、統合された認証情報プロバイダー経由で取得します。

```python
from bedrock_agentcore.identity.auth import requires_api_key, requires_access_token

@requires_api_key(provider_name="MyAPI")
def call_api(payload: dict, *, api_key: str) -> dict:
    ...

@requires_access_token(provider_name="MyOAuthProvider", scopes=["read"], auth_flow="M2M")
async def call_downstream(data: dict, *, access_token: str) -> dict:
    ...
```

デコレーターは、呼び出し時に Secrets Manager から取得し、キャッシュと更新を処理します。この方法で登録された認証情報は、保存時に暗号化され、再デプロイなしでローテーションできます。

**ローカル開発:** `agentcore/.env.local`(gitignore 対象)を `agentcore dev` が読み取るので、デコレーターはローカルで解決されます。このファイルは、デプロイ時にランタイムへアップロード**されません**。本番の認証情報は、認証情報プロバイダーにあります。

---

## ツールの範囲: エージェントのコードでの HTTP 直接呼び出しより、Gateway ターゲットを優先する

関連する監査です。エージェントが呼び出す外部サービスごとに、エージェントのコードに埋もれた HTTP の直接呼び出しではなく、Gateway ターゲットにすべきではないかを問います。Gateway の認証情報プロバイダーは、エッジで認証を注入するため(エージェントのプロセスがシークレットを見ることはありません)、ツールのカタログにポリシーを適用でき、エージェントのコードから漏れたトレースバックやログの 1 行から、そもそも届いていない認証情報が流出することもありません。

```bash
# エージェントのコードから、アウトバウンドの HTTP 直接呼び出しを探す
grep -rEn 'httpx\.|requests\.|aiohttp\.' app/ --include="*.py"
```

ヒットごとに、次のように判断します。

| ヒットの内容 | アクション |
|---|---|
| エージェントがツールとして扱う外部の REST API を呼び出している | Gateway ターゲットとして前段に置く(`agentcore add gateway-target --type open-api-schema` または `api-gateway`)。[`agents-connect/SKILL.ja.md`](../agents-connect/SKILL.ja.md) の Path C を読み込む。 |
| MCP サーバーを直接呼び出している | Gateway ターゲットとして前段に置く(`--type mcp-server`)。[`agents-connect/SKILL.ja.md`](../agents-connect/SKILL.ja.md) の Path A を読み込む。 |
| AWS サービス(S3、DynamoDB など)を呼び出している — この行に当てはまるのは不適切で、`boto3` であるべき | `requests`/`httpx` から `boto3` のクライアントに移行し、IAM にはランタイムの実行ロールを使う。認証情報は不要。 |
| ストリーミングサービス(ライブ出力のある SSE、WebSocket、WebRTC)を呼び出している | 直接のままで問題ない — Gateway はまだこれらを前段に置けない。認証が `os.getenv` ではなく、`@requires_*` を使っていることを確認する。 |
| A2A で別のエージェントを呼び出している | 直接のままで問題ない — A2A は設計上 HTTP である。Bearer トークンに `@requires_access_token` を使っていることを確認する。 |
| 測定済みのレイテンシーのホットパスを呼び出しており、チームがそれを選択した | 問題ないが、測定が存在し、認証が `@requires_*` を使っていることを確認する。 |

ヒットが、「直接のままで問題ない」の行のどれにも当てはまらない場合は、Gateway ターゲットに変換するチケットを起票します。ほとんどのフレームワーク統合では、エージェントのコードを変更せずに Gateway ターゲットを追加できます(MCP のツール検出がバインドを処理します)。

---

## オブザーバビリティ: トレーシングが有効であることを確認する

AgentCore は、X-Ray のトレーシングと CloudWatch のロギングを自動で有効にします。確認します。

```bash
agentcore status --runtime <AgentName> --json | jq '.runtimes[0].observabilityConfig'
```

**CloudWatch のダッシュボード:** AWS コンソール → CloudWatch → GenAI Observability → Bedrock AgentCore

**ログの保持期間:** デフォルトでは、ログは無期限に保持されます。コスト管理のために、保持ポリシーを設定します。

```bash
aws logs put-retention-policy \
  --log-group-name /aws/bedrock-agentcore/runtimes/<AGENT_ID>-DEFAULT \
  --retention-in-days 30
```

---

## 評価のベースライン: ローンチ前に品質を把握する

本番に移行する前に、品質のベースラインを確立して、回帰を検出できるようにします。

```bash
# ベースラインの評価を実行する
agentcore run eval \
  --evaluator "Builtin.Helpfulness" \
  --evaluator "Builtin.GoalSuccessRate"

# 継続的なモニタリングをセットアップする
agentcore add online-eval \
  --name production_monitor \
  --runtime <AgentName> \
  --evaluator "Builtin.Helpfulness" \
  --sampling-rate 5
agentcore deploy -y
```

ベースラインのスコアを記録します。変更の後にスコアが大きく下がった場合は、続行する前に調査してください。

---

## ネットワーク: プライベートリソース向けの VPC

エージェントがプライベートな AWS リソース(RDS、内部 API)にアクセスする場合は、VPC を設定します。

```bash
agentcore add agent \
  --name MyAgent \
  --network-mode VPC \
  --subnets subnet-abc,subnet-def \
  --security-groups sg-123
```

VPC の設定の完全なガイダンスは、`agents-build`([`references-ja/vpc.md`](../agents-build/references-ja/vpc.md) を読み込む)を参照してください。

---

## 初期化時間: コールドスタートのパフォーマンスを最適化する

エージェントの初期化が遅いと、タイムアウト、424 エラー、ユーザー体験の低下を招きます。特に、しばらく使われなかった後の最初の呼び出しで顕著です。エージェントがリクエストを処理できる状態になる前に行うことは、すべてユーザーの待ち時間に加算されます。

### コールドスタートの時間が実際にどこに使われるか

新しい環境の一般的なコールドスタートは、20〜30 秒程度かかります。大まかな内訳は次のとおりです。

- **コンテナイメージの取得** — Container ビルドで支配的です。100 MB のイメージなら数秒、500 MB のイメージなら 15 秒以上かかることがあります。
- **アプリケーションの起動** — コードのインポート時間、フレームワークの初期化、モジュールレベルのセットアップ。通常は 5〜10 秒で、インポート時にモデルを読み込んだり接続を開いたりすると、はるかに長くなることがあります。
- **プラットフォームのオーバーヘッド**(microVM の起動、ネットワークのアタッチ、コンテナの起動)— 1 秒未満から数秒。

自分で制御できるのは、イメージのサイズとアプリケーションの起動の 2 つです。どちらを最適化しても、最初の応答までの時間が直接短縮されます。

### セッションの再利用が、最も効果の高い最適化

同じセッションのリクエストは、すでに初期化された既存の環境にルーティングされるので、コールドスタートはありません。セッションごとの最初のリクエストがコールドスタートのコストを支払い、同じセッションのそれ以降のリクエストはすべて高速です。

具体的なパターン:

- **マルチターンの会話:** ターンをまたいで、同じ `session_id` を再利用する。ターンごとに新しい UUID を生成しない。
- **バッチ処理:** バッチ内の項目をまたいで、同じ `session_id` を再利用する。
- **ユーザー向けのアプリ:** セッションを、メッセージごとではなく、ユーザーの操作の単位(例: チャットの会話ごとに 1 セッション)にスコープする。

SDK 間の注意: MCP を使っている場合は、`runtimeSessionId` と `mcpSessionId` の両方ではなく、**どちらか 1 つの**セッション識別子を渡してください。両方を送ると、プラットフォームが同じ論理セッションに 2 つの別々の環境をバインドし、コールドスタートのコストが 2 倍になることがあります。

### パッケージサイズの予算

デプロイパッケージの 1 MB ごとに、コールドスタートの時間が増えます。

- **目標:** 200 MB 未満。可能なら 100 MB 未満を目指す。
- **Container ビルドの場合:** マルチステージの Dockerfile、slim または distroless のベースイメージを使い、ビルドツールとテストファイルを取り除き、`.dockerignore` を追加する。
- **CodeZip ビルドの場合:** `pyproject.toml` / `requirements.txt` から開発用の依存関係を取り除く。`tests/`、`docs/`、`.git/`、ローカルのキャッシュを含めない。
- **定期的に監査する:** `pip list`(Python)や `npm ls`(Node)で、実際にインストールされているものを確認できる。使っていないものを削除する。

### 重い初期化を後回しにする

大きなモデルの読み込み、データベースへの接続、MCP クライアントの初期化を、モジュールのインポート時に行わないでください。モジュールのインポートに費やす 1 秒は、エージェントがリクエストに応答できない 1 秒です。

```python
# ❌ 遅い — インポート時に実行され、エージェントがリクエストを処理できるようになる前に走る
import heavy_library
client = heavy_library.Client(config)

# ✅ 速い — 最初のリクエストまで遅延させる
_client = None
def get_client():
    global _client
    if _client is None:
        import heavy_library
        _client = heavy_library.Client(config)
    return _client
```

### デプロイの種類は、デフォルトではなく、トラフィックのパターンに基づいて選ぶ

このスキルは以前、可能なら Container よりも CodeZip を推奨していました。これは単純化しすぎです。実際のトレードオフは次のとおりです。

- **CodeZip:** 反復が簡単で、表面積が小さい。コールドスタートには、コードのダウンロードと展開が含まれます — 約 95 MB のパッケージでは、アプリケーションの起動が始まる前に、プラットフォームのダウンロードで約 1.3 秒が加わります。
- **Container:** イメージ全体を自分で制御でき、独自のシステム依存関係が必要な場合に必要です。大きなイメージはコールドスタートごとのコストが高くなりますが、マルチステージビルドで積極的に最適化できます。

どちらかが常に勝つわけではありません。どちらも、セッションの再利用と、パッケージを小さく保つことから、同じように恩恵を受けます。トラフィックのパターンに、バースト的なコールドセッションが多い場合は、使っているデプロイのアーティファクトを小さくすることに投資してください。トラフィックのパターンでセッションが再利用されるなら、デプロイの種類はほとんど重要ではありません。

### Gateway の背後にある Lambda ターゲットの場合

Lambda 関数のプロビジョニング済み同時実行を使って、Lambda のコールドスタートをなくします。これは Runtime の初期化とは別のものです。コールドな Lambda の最初の呼び出しでレイテンシーが加わるのは、Lambda 自体です。

---

## セッションのライフサイクル管理

セッションの管理は、コスト、パフォーマンス、`maxVms` のクォータと密接に関連しています。これを正しく行えるかどうかが、スムーズな本番ローンチと、クォータに阻まれるローンチを分けることも多いです。

### デフォルトのライフサイクル

新しいセッション ID のリクエストが届くと、ランタイムはそのために新しい環境を初期化します。その環境は、次のいずれかが起きるまで稼働し続けます。

1. **`StopRuntimeSession` で、セッションが明示的に停止される。**
2. **アイドルタイムアウトが切れる。** ランタイムは、`idleRuntimeSessionTimeout`(デフォルトは 900 秒)の間リクエストを受け取らなかった環境を回収します。
3. **最大の存続時間に達する**(`maxLifetime`、デフォルトは 8 時間)。

アイドル状態の環境は、トラフィックを処理していなくても、回収されるまで `maxVms` のクォータにカウントされます。これが、予期しない `maxVms` エラーの第 1 の原因です。

### ワークロードの形に合わせてタイムアウトを選ぶ

本番では、デフォルトのままにしないでください。ワークロードが実際にセッションをどう使うかに合った値を選びます。

| ワークロード | `idleRuntimeSessionTimeout` | `maxLifetime` | 理由 |
|---|---|---|---|
| 対話型のチャット / サポートエージェント | 600〜900 秒(デフォルト) | 3600〜7200 秒 | ユーザーは、読んだり考えたりするために間を置く。離脱したら、早めに回収する。 |
| フォローアップのない、リクエスト/応答の API | 60〜120 秒 | 1800 秒 | 各呼び出しが自己完結している — VM をすぐに解放する。 |
| バッチ処理、ジョブごとに 1 セッション | 120 秒 | ジョブの長さ + バッファ | バッチ内の項目間のアイドルの間隔は小さい。ジョブ間では積極的に回収する。 |
| バックグラウンド / 長時間実行のタスク(`add_async_task` を使用) | 120〜300 秒 | 最大 28800 秒(8 時間) | 非同期タスクの API は、追跡中の作業の間、VM を稼働させ続ける。アイドルタイムアウトは、タスクの合間に適用される。 |

**トレードオフの概要:**

- **アイドルタイムアウトが短い** = `maxVms` に対する余裕が増え、コストが下がる。**リスク:** 会話の途中で回収され、次のターンがコールドスタートになる。
- **アイドルタイムアウトが長い** = ウォームなターンで、レイテンシーが低い。**リスク:** アイドルの VM がクォータを消費し、バーストで `maxVms` のエラーが出る。
- **最大の存続時間が短い** = 予測可能な再生成で、メモリリークや古い状態を抑えられる。**リスク:** アクティブな長いセッションが、途中で終了させられる。
- **最大の存続時間が長い** = セッションが維持され、ウォーム状態の節約が大きい。**リスク:** ドリフト、メモリ内の古い状態、ロールアウトの難しさ。

### ベストプラクティス

**作業が終わったら、`StopRuntimeSession` を呼び出す。** エージェントがタスクを終え、そのセッションでこれ以上のリクエストを想定していない場合は、明示的に停止します。これにより、アイドルタイムアウトを待たずに、環境がすぐに解放されます。

```python
# 呼び出しのロジックが完了し、セッションが終わったと分かっている時点で:
client.stop_runtime_session(
    agentRuntimeArn=runtime_arn,
    runtimeSessionId=session_id,
)
```

**関連する作業には、セッション ID を再利用する。** HTTP リクエストごとに新しいセッション ID を使うと、HTTP リクエストごとに新しい環境ができます。マルチターンの会話、バッチジョブ、ユーザー向けの操作では、会話/バッチ/ユーザーの操作ごとに 1 つのセッション ID を使い、関連するすべてのリクエストをそこにルーティングします。

**`idleRuntimeSessionTimeout` を、ワークロードに合わせて調整する。** デフォルトの 900 秒は、すぐにフォローアップのリクエストが来ると想定される、対話型のワークロードに適しています。セッションが短命な、リクエスト/応答のワークロードでは、短くしてください。

`agentcore/agentcore.json` のランタイムのエントリを編集します。

```json
{
  "runtimes": [
    {
      "name": "MyAgent",
      "lifecycleConfiguration": {
        "idleRuntimeSessionTimeout": 120,
        "maxLifetime": 3600
      }
    }
  ]
}
```

その後、`agentcore deploy` で適用します。CLI と CDK が、その背後の `UpdateAgentRuntime` の呼び出しを処理します。

CLI を使いたい場合は、`agentcore add agent ... --idle-timeout 120 --max-lifetime 3600` で、同じフィールドが `agentcore.json` に書き込まれます。このファイルが唯一の正しい情報源です。ファイルの先頭にある `$schema` の URL(`https://schema.agentcore.aws.dev/v1/agentcore.json`)により、すべてのフィールドで IDE のオートコンプリートが使えます。

タイムアウトが短い = VM の回収が速い = `maxVms` に対する余裕が増える。短すぎると、会話の途中で環境が回収され、次のターンがコールドスタートになります。

**`runtimeSessionId` と `mcpSessionId` を一緒に渡さない。** MCP のエージェントでは、どちらか一方を使います。両方を渡すと、同じ論理セッションに 2 つの別々の VM がバインドされることがあります。

### `maxVms` の問題を診断する

`ServiceQuotaExceededException: maxVms limit exceeded` が発生した場合でも、最初にクォータの引き上げを申請しないでください。CloudWatch の同時セッションのメトリクスは、稼働中の VM の数とは異なります。アイドル状態の環境は、回収されるまでクォータにカウントされます。

次の順で進めます。

1. 各論理リクエストの完了後に、`StopRuntimeSession` を追加する
2. セッション ID の生成を監査する — 再利用すべきなのに、リクエストごとに新しい ID を作っていないか
3. セッションが短命なら、`idleRuntimeSessionTimeout` を下げる
4. 上記をすべて行ってもなお上限に当たる場合に限って、引き上げを申請する

引き上げの申請のワークフロー(Service Quotas コンソール経由)と、根拠のテンプレートは、[`references-ja/limits.md`](references-ja/limits.md) を参照してください。

---

## 長時間実行のバックグラウンドタスク

エージェントが、`/invocations` のレスポンスより長く続く作業(バックグラウンド処理、非同期ジョブ、長いツールチェーン)を起動する場合、投げっぱなしのパターンでは不十分です。呼び出しのレスポンスが送信されると、ランタイムはセッションをアイドルと見なすため、バックグラウンドのタスクがまだ実行中でも、環境が `idleRuntimeSessionTimeout` で回収されることがあります。

### SDK の非同期タスク API で、「まだビジー」であることを伝える

bedrock-agentcore SDK は、追跡対象の作業が実行されている間、環境を稼働させ続けるタスク登録を提供します。Python では:

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload, context):
    # タスクを開始する前に、先に登録する
    task_id = app.add_async_task("background_work")

    # 作業を開始する（スレッド、asyncio などで）
    start_background_work(task_id, payload)

    # 呼び出しのレスポンスを返す — タスクは引き続き追跡される
    return {"status": "processing", "taskId": task_id}


def start_background_work(task_id, payload):
    try:
        # ここで長時間の作業を行う
        do_the_work(payload)
    finally:
        # 終わったらタスクを完了としてマークする — これで「ビジー」のシグナルが解除される
        app.complete_async_task(task_id)

if __name__ == "__main__":
    app.run()
```

登録されたタスクが少なくとも 1 つアクティブな間は、ランタイムは環境をビジーと見なし、`idleRuntimeSessionTimeout` で回収しません。`maxLifetime`(デフォルトは 8 時間)は、引き続き厳格な上限として適用されます。

お使いの言語で同等の API については、bedrock-agentcore SDK のドキュメントを確認してください。TypeScript SDK にも類似のパターンがあります。

### 非同期タスクの API が使えない場合の代替手段

- **`idleRuntimeSessionTimeout` を、想定されるタスクの所要時間に合わせて長くする。** タスクが最大 10 分実行されると分かっているなら、タイムアウトを 12 分にします。`maxLifetime` よりは十分に短く保ってください。
- **ストリーミングレスポンスで HTTP 接続を開いたままにし、定期的にハートビートのイベントを送出する。** 呼び出し元に、ポーリングではなく結果を待たせたい場合に役立ちます。[`agents-debug/SKILL.ja.md`](../agents-debug/SKILL.ja.md)(「ストリーミング接続がレスポンスの途中で切れる」のセクション)の SSE キープアライブのパターンを参照してください。
- **長い作業を、同じセッション上の複数の呼び出しに分割する。** 各呼び出しで、アイドルのタイマーがリセットされます。

---

## クォータと上限

スロットリング、`ServiceQuotaExceededException`、その他のクォータ関連のエラーが発生している場合、またはローンチを控えていて、クォータが妨げにならないようにしたい場合は、[`references-ja/limits.md`](references-ja/limits.md) を読み込んでください。

このリファレンスが扱う内容:

- 各エラーがどのクォータに対応するか
- 引き上げを申請する前に試す軽減策(重要 — 「クォータ」のエラーの大半は、実際にはセッションのライフサイクルの問題です)
- Service Quotas コンソールで引き上げを申請する方法(サポートに直接ケースを起票する必要があるのは、まれな例外的なケースです)
- 審査担当者が承認するために必要なものがすべて入った、コピー&ペーストできる根拠のテンプレート

---

## 本番準備のチェックリストの概要

プロジェクトに固有のチェックリストを生成します。

```
Production Readiness Checklist for <AgentName>

IAM
[ ] Execution role Bedrock access scoped to specific model ARNs
[ ] ECR access scoped to specific repository
[ ] Trust policy scoped to your account ID

Authentication
[ ] Inbound auth is AWS_IAM or CUSTOM_JWT (not NONE)
[ ] If CUSTOM_JWT: discovery URL, audience, and client IDs configured

Shell Access (if using InvokeAgentRuntimeCommand)
[ ] InvokeAgentRuntimeCommand permission granted only to identities that need it
[ ] Separate IAM policy from InvokeAgentRuntime policy
[ ] CloudTrail / EventBridge alert configured for InvokeAgentRuntimeCommand calls
[ ] If commands constructed from user input: shell injection validation implemented

Code quality
[ ] Error handling wraps all agent logic
[ ] Input validation on payload fields (type, length, format)
[ ] No secrets hardcoded in agent code
[ ] Credentials registered via agentcore add credential

Observability
[ ] X-Ray tracing enabled (auto-configured)
[ ] CloudWatch log retention policy set
[ ] Eval baseline established

Performance
[ ] Agent initialization time measured and optimized
[ ] Deployment package size under 200 MB (target under 100 MB)
[ ] Dependencies audited — no unused packages
[ ] Heavy initialization deferred to request time
[ ] Session reuse strategy chosen for multi-turn / batch workloads
[ ] `StopRuntimeSession` called after work completes where applicable
[ ] `idleRuntimeSessionTimeout` tuned to workload (default 900s)
[ ] For long-running background tasks: `add_async_task` / `complete_async_task` used

Resources
[ ] Memory strategies appropriate for use case (if using memory)
[ ] Gateway auth configured (if using gateway)
[ ] Policy engine attached (if restricting tool access)

Testing
[ ] Agent tested with production-representative inputs
[ ] Error cases tested (tool failures, model errors)
[ ] Memory cross-session tested (if using LTM)
```

## 出力

- プロジェクトに固有の所見を含むチェックリスト
- 見つかった問題を修正するための具体的なコマンド
- 検出したモデルとリソースに対する、推奨の IAM ポリシー
