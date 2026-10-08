# エージェントのバージョン管理とロールバック

`agentcore deploy` を実行するたびに、エージェントランタイムの新しいバージョンが作成されます。この資料では、バージョンの仕組み、特定のバージョンへの固定、デプロイで問題が起きたときのロールバックを扱います。

## バージョン管理の仕組み

- `agentcore deploy` のたびに、新しいランタイムのバージョンが作成される
- エイリアス（通常は `DEFAULT`）が、現在ライブのバージョンを指す
- 古いバージョンは、ロールバック用に ARN でアクセスできるまま残る
- ローカル開発（`agentcore dev`）は、常に現在のコードを実行する — バージョンの概念はない

## バージョンを調べる

現在の AgentCore CLI が管理するのは、プロジェクトの設定（`agentcore.json` → `agentcore deploy`）で、バージョンやエイリアスの操作は直接公開していません。それらについては、`bedrock-agentcore-control` のコントロールプレーンに対して、AWS CLI を使います。

```bash
# エージェントのすべてのバージョンを一覧表示する
aws bedrock-agentcore-control list-agent-runtime-versions \
  --agent-runtime-id <AGENT_RUNTIME_ID>

# 特定のバージョンの詳細を取得する
aws bedrock-agentcore-control get-agent-runtime \
  --agent-runtime-id <AGENT_RUNTIME_ID> \
  --qualifier <VERSION>
```

`<AGENT_RUNTIME_ID>` は、`agentcore status --json | jq '.runtimes[0].agentRuntimeId'` で取得できます。

## 特定のバージョンを呼び出す

デフォルトでは、呼び出し元はエイリアス（現在のバージョン）にアクセスします。呼び出しを特定のバージョンに固定するには、呼び出しリクエストで `qualifier` を渡します。

```python
response = client.invoke_agent_runtime(
    agentRuntimeArn="<AGENT_RUNTIME_ARN>",
    qualifier="3",  # バージョン 3 を指定して呼び出す
    payload=payload,
    runtimeSessionId=session_id,
)
```

これは、次のような場合に役立ちます。

- カナリアテスト — 切り替えの前に、トラフィックのごく一部を新しいバージョンに送る
- A/B 比較 — 2 つのバージョンを並行して実行し、出力を比較する
- デバッグ — 特定のバージョンで問題を再現する

## ロールバック

デプロイで何かが壊れた場合は、以前の正常なコードを再デプロイすることでロールバックします。

```bash
# オプション 1: 以前のコミットを git checkout して、再デプロイする
git checkout <PREVIOUS_COMMIT>
agentcore deploy -y

# オプション 2: エイリアスを古いバージョンに向ける（コードのロールバックは不要）
aws bedrock-agentcore-control update-agent-runtime-alias \
  --agent-runtime-id <AGENT_RUNTIME_ID> \
  --alias-name DEFAULT \
  --routing-configuration agentRuntimeVersion=<OLDER_VERSION>
```

オプション 2 のほうが高速です。再ビルドも再デプロイも不要で、ポインターを切り替えるだけです。オプション 1 は、コードが稼働中のものと一致するため、よりクリーンです。

## カナリアデプロイ

2 つのバージョン間でトラフィックを分割し、全面展開の前に新しいデプロイを検証します。

```bash
aws bedrock-agentcore-control update-agent-runtime-alias \
  --agent-runtime-id <AGENT_RUNTIME_ID> \
  --alias-name DEFAULT \
  --routing-configuration \
    agentRuntimeVersion=<NEW_VERSION>,weight=10 \
    agentRuntimeVersion=<OLD_VERSION>,weight=90
```

これは、トラフィックの 10% を新しいバージョンにルーティングします。重みを増やす前に、`agents-optimize` の評価スコアとエラー率を監視してください。

## バージョンのクリーンアップ

AgentCore はバージョンを無期限に保持します。自動では削除されません。数百回デプロイした場合は、古いバージョンを定期的に削除することを検討してください。

```bash
aws bedrock-agentcore-control delete-agent-runtime-version \
  --agent-runtime-id <AGENT_RUNTIME_ID> \
  --version <OLD_VERSION>
```

現在ライブのバージョンは、決して削除しないでください。

## ステージングターゲット

dev/staging/prod の環境を分けたいチームは、デプロイターゲットを使います。

```json
// agentcore/aws-targets.json
[
  {"name": "default", "account": "<DEV_ACCOUNT>", "region": "us-east-1"},
  {"name": "staging", "account": "<STAGING_ACCOUNT>", "region": "us-east-1"},
  {"name": "production", "account": "<PROD_ACCOUNT>", "region": "us-west-2"}
]
```

```bash
agentcore deploy --target staging -y
agentcore deploy --target production -y
```

各ターゲットは、独自のランタイムを持ちます。バージョンはターゲットごとに別々です。

## 相互参照

- 特定の障害のためにロールバックが必要な場合は、先に `agents-debug` で診断してください
- ステージング/本番のベストプラクティスは、`agents-harden` を参照してください
- 切り替えの前に、特定のバージョンに対して評価を実行するには、[`agents-optimize/references-ja/evals.md`](../../agents-optimize/references-ja/evals.md) を参照してください
