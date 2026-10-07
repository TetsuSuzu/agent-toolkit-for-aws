# ローカルとデプロイ済み環境 — どこで何が動くか

AgentCore には、ローカル開発サーバー（`agentcore dev`）とデプロイ済みのランタイムがあります。両者の機能は同等ではありません。この資料は、どこで何が動くかをまとめたもので、生成するコードやトラブルシューティングが両方の環境を正しく扱えるようにするためのものです。

## クイックリファレンス

| 機能 | `agentcore dev`（ローカル） | デプロイ済みランタイム |
|---|---|---|
| エージェントの呼び出し | ✅ localhost:8080 に curl | ✅ `invoke_agent_runtime` または HTTPS 経由 |
| フレームワークのモデル呼び出し | ✅ Bedrock の認証情報があれば可 | ✅ |
| Python/JS の関数ツール（フレームワークネイティブ） | ✅ | ✅ |
| 認証情報（`@requires_api_key`、`@requires_access_token`） | ✅ `agentcore/.env.local` から | ✅ Secrets Manager から |
| Memory | ❌ ローカルでは環境変数が設定されない | ✅ `MEMORY_<NAME>_ID` が注入される |
| Gateway | ❌ ローカルでは環境変数が設定されない | ✅ `AGENTCORE_GATEWAY_<NAME>_URL` が注入される |
| Cedar ポリシーの評価 | ❌ ポリシーは Gateway でのみ適用される | ✅ |
| トレース（X-Ray） | ✅ `agentcore dev` はデフォルトで OTEL を CloudWatch に出力する。`--no-traces` で無効化 | ✅ 自動で有効 |
| CloudWatch Logs | ✅ ADOT / OTEL の接続経由（トレースと同じ経路） | ✅ `logging` モジュール + OTEL を使う場合 |
| **Evaluator の*定義***（`agentcore add evaluator`、指示文やコードの記述） | ✅ — `agentcore.json` に書き込まれる。カスタムコードはローカルでユニットテスト可能 | ✅ |
| **`agentcore run eval`**（トレースに対するオンデマンド評価） | ✅ — CloudWatch のスパンに対して動作する。OTEL が有効（デフォルト）ならローカル開発のスパンもそこに届く | ✅ |
| **手作りのスパンによる `Evaluate` API**（boto3） | ✅ — ランタイムは一切不要。`SessionSpans` を直接送信する | ✅ |
| **データセットランナー**（`OnDemandEvaluationDatasetRunner`） | ❌ パイプライン内で AgentCore Runtime のエージェントを呼び出す | ✅ |
| **オンライン評価のモニタリング**（`agentcore add online-eval`） | ❌ デプロイ済みランタイムからトレースを継続的に取り込む | ✅ |
| オブザーバビリティのダッシュボード | ✅ Transaction Search が有効で、ローカルのスパンが流れていれば可 | ✅ CloudWatch コンソール |
| VPC ネットワーク | ❌ ローカルは常にインターネットに接続できる | ✅ `networkMode: VPC` に従う |
| インバウンド認証（AWS_IAM、CUSTOM_JWT） | ❌ ローカルでは認証不要 | ✅ すべてのリクエストで強制される |

## 生成するコードへの影響

**ローカルで使えない機能は、必ずガードしてください。**

```python
# Memory のパターン
MEMORY_ID = os.getenv("MEMORY_MYMEMORY_ID")
if MEMORY_ID:
    # デプロイ済み — Memory を接続する
    session_manager = AgentCoreMemorySessionManager(...)
else:
    # ローカル — Memory なしでエージェントを実行する
    session_manager = None
```

```python
# Gateway のパターン
GATEWAY_URL = os.getenv("AGENTCORE_GATEWAY_WEATHER_URL")
if GATEWAY_URL:
    # デプロイ済み — Gateway のツールを使う
    tools = get_gateway_tools(GATEWAY_URL)
else:
    # ローカル — 外部ツールなし、またはローカルのスタブで実行する
    tools = []
```

**認証情報は両方の環境で動作しますが、読み取り元が異なります。** `@requires_api_key` デコレーターがこれを自動的に処理します。環境変数を直接読もうとしないでください。

## テストの流れ

Memory、Gateway、ポリシーはローカルでは動作しないため、現実的なテストの流れは次のとおりです。

1. **ローカル:** `agentcore dev` で、エージェントのコード構造、フレームワークの接続、システムプロンプト、コード内のロジックを確認します
2. **ステージング環境へデプロイ:** `agentcore deploy --target staging` で、実際の Memory、Gateway、ポリシーを使ってテストします
3. **本番:** ステージングでの検証後にのみ行います

Memory の想起、Gateway のツール呼び出し、ポリシーによる拒否に関わる本番の障害は、`agentcore dev` では再現できません。デプロイ済みの環境が必要です。

## 「ローカルでは動くのにデプロイすると失敗する」よくある原因

- `MEMORY_<NAME>_ID` のガードがない — 環境変数が想定外に存在するため、コードがクラッシュする
- Gateway にハードコードした localhost の URL — `AGENTCORE_GATEWAY_<NAME>_URL` に置き換える
- 開発用の認証情報では動くが、実行ロールでは動かない IAM 権限
- `aws configure`（ローカルで使用）と `aws-targets.json`（デプロイで使用）のリージョンの不一致
- 個人の認証情報では動くが、実行ロールからの Gateway SigV4 では動かないツール呼び出しの認証

## 「デプロイすると動くのにローカルでは失敗する」よくある原因

- Memory や Gateway の環境変数が常に設定されていると仮定したコード
- デプロイ済みの実行ロールの権限を前提とした SDK の直接呼び出し
- デプロイ環境専用の URL や ARN のハードコード
