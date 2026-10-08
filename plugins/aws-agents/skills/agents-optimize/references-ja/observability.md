# オブザーバビリティのセットアップ

AgentCore のエージェントに、ロギング、トレーシング、モニタリングを設定します。

## 自動で有効になるもの

AgentCore は、次のものを自動的に有効にします。

- **X-Ray トレーシング** — すべての呼び出しがトレースを生成する
- **CloudWatch ロギング** — エージェントのログが CloudWatch に送られる

これらは、**デプロイ済み**(`agentcore deploy` + 呼び出し)でも、**ローカル**(`agentcore dev`)でも、デフォルトでオンです。開発サーバーは、デプロイ済みのランタイムと同じように、AWS OpenTelemetry ディストリビューションでエージェントを自動的に計装します。オプトアウトするには、`agentcore dev --no-traces` を使います。

ローカルの経路がエンドツーエンドで動作するための前提条件が 2 つあります。

1. **ローカルで AWS の認証情報が使えること** — OTEL のエクスポーターがスパンを送るために必要です。
2. **アカウントで CloudWatch Transaction Search が有効になっていること**(アカウントごとに一度だけの設定)— 下の「トレースを表示する」を参照してください。これがないと、スパンは取り込まれても検索できないため、`agentcore traces list` と `agentcore run eval --session-id` は空の結果を返します。

デプロイ後は、AgentCore Runtime がコンテナも自動的に計装します(デフォルトの CMD が、アプリを `opentelemetry-instrument` でラップします)。どちらの経路でも、コードで OTEL を設定する必要はありません。ただし、エージェントのコードが正しく計装されている必要はあります。

## ログが CloudWatch に表示されるようにする

ログが表示されるには、次の 3 つが満たされている必要があります。

### 1. Dockerfile の OTEL エントリーポイントのラッパー

Dockerfile の CMD で、OpenTelemetry のラッパーを使う必要があります。

```dockerfile
CMD ["opentelemetry-instrument", "python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

CodeZip ビルドでは、これは自動で処理されます。Container ビルドでは、自分で追加する必要があります。

### 2. CloudWatch と X-Ray の IAM 権限

ランタイムの実行ロールには、次のものが必要です。

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "xray:PutTraceSegments",
    "xray:PutTelemetryRecords"
  ],
  "Resource": "*"
}
```

### 3. `print()` ではなく、`logging` モジュールを使う

AgentCore は、Python の `logging` モジュールを通じて、構造化されたログを取得します。`print()` の出力は stdout に出ますが、OTEL のパイプラインでは取得されません。

```python
import logging
logger = logging.getLogger(__name__)

# 良い例 — CloudWatch に取得される
logger.info("Processing request", extra={"session_id": session_id})

# 悪い例 — 取得されない
print(f"Processing request {session_id}")
```

## トレースを表示する

トレースは、1 回のエージェントの呼び出しの完全な実行経路(モデル呼び出し、ツール呼び出し、タイミング)を示します。

```bash
# 最近のトレースを一覧表示する
agentcore traces list --runtime <AgentName> --since 1h --limit 10

# 特定のトレースを取得する
agentcore traces get <traceId> --runtime <AgentName>
```

**トレースの遅延:** トレースは、呼び出しの**約 10 秒後**に表示されます(以前は 30〜60 秒でした)。すぐに表示されなくても慌てないでください。また、スクリプトに長い待ち時間を組み込まないでください。「30〜60 秒」や「2〜5 分」と書かれた古いスキルやドキュメントは、情報が古くなっています。

また、CloudWatch で **Transaction Search** が有効になっていることも確認してください。これは、コンソールでトレースを表示するための前提条件です。

## ログを表示する

```bash
# 最近のログをストリーミングする
agentcore logs --runtime <AgentName> --since 30m

# レベルで絞り込む
agentcore logs --runtime <AgentName> --level error --since 1h

# 特定のテキストを検索する
agentcore logs --runtime <AgentName> --query "timeout" --since 2h
```

## CloudWatch のダッシュボード

本番のエージェントには、次の内容を含む CloudWatch のダッシュボードを設定します。

- 呼び出し回数とエラー率
- P50/P90/P99 のレイテンシー
- メモリと CPU の使用率
- 種類別のエラーログ数

これらのメトリクスは、デプロイ後に `AWS/BedrockAgentCore` 名前空間で利用できます。

## マルチアカウントのオブザーバビリティ

エージェントが複数のアカウントに分散している場合(典型的な構成は、prod / staging / dev のアカウントを分けるもの)は、**CloudWatch のクロスアカウントオブザーバビリティ**を使って、1 つの中央のモニタリングアカウントからメトリクス、トレース、ログを表示します。

設定の順序が重要です。この順序で行わないと、コンソールにソースアカウントのデータが表示されません。

1. **モニタリングアカウントを選ぶ。** すべてを表示する場所です。多くの場合、ワークロードのアカウントではなく、中央のオブザーバビリティ用のアカウントです。
2. **先にモニタリングアカウントを設定する。** CloudWatch コンソール → Settings → Monitoring account configuration → Configure。共有するテレメトリの種類を選びます(メトリクスと**ログ**の両方を有効にします — トレースは X-Ray 独自のクロスアカウントの仕組みを通ります)。
3. **各ソースアカウントをリンクする。** AWS Organizations 経由(アカウントが 1 つの組織内にある場合)か、個別のリンクで行います。ソースアカウントは、リンクを承認する必要があります。
4. **オブザーバビリティを有効にして、ソースアカウントに AgentCore のエージェントをデプロイする。** 単一アカウントの場合と同じ、デフォルトの OTEL ラップを使います。コードの変更は不要です。
5. **モニタリングアカウントから表示する。** CloudWatch コンソールの AgentCore Observability に、リンクされたすべてのアカウントのデータが、ソースアカウント ID で識別されて並んで表示されます。

**順序の落とし穴:** リンクの*前*にソースアカウントにエージェントをデプロイした場合でも、テレメトリは正しく流れます。ただし、リンクが有効になるまでは、モニタリングアカウントから見えません。再デプロイは不要で、リンクの後に数分待つだけです。

**トレース:** クロスアカウントでのトレースの表示は、X-Ray の既存のクロスアカウント共有モデルを使います。ログとメトリクスの CloudWatch のクロスアカウントリンクが正しく設定されているのに、トレースが表示されない場合は、X-Ray のクロスアカウント設定を別途確認してください。

**IAM:** クロスアカウントのオブザーバビリティのために、エージェントの実行ロールに追加の IAM は不要です。クロスアカウントの機能は、テレメトリの送信元ではなく、CloudWatch/X-Ray のレイヤーで動作します。

現在のコンソールの操作と、エッジケースについては、[クロスアカウントのオブザーバビリティ](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-cross-account.html)を参照してください。

## 相互参照

- ログがまったく表示されない場合は、上の 3 つの要件を確認するか、`agents-debug` を使ってください
- 本番のオブザーバビリティのセットアップについては、`agents-harden` を参照してください
- (運用上の健全性だけでなく)エージェントの品質を測定するには、[`references-ja/evals.md`](evals.md) を読み込んでください
