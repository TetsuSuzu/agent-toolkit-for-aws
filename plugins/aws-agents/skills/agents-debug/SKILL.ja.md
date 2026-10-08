---
name: agents-debug
description: >
  エージェントや環境が壊れているときに使います — 誤った回答、エラー、タイムアウト、
  ツールの失敗、CLI の問題など。トレースとログを読んで、根本原因を診断します。
  CLI 自体が動かない場合の前提条件の確認も行います。トリガー: "agent not working"、"wrong answer"、
  "agent error"、"tool call failing"、"debug agent"、"check logs"、
  "read traces"、"broken"、"500 error"、"424 error"、"model access
  denied"、"command not found"、"stuck in DELETING"、"maxVms exceeded"、
  "cold start diagnosis"、"cold start slow"、"agentcore create error"、
  "create failed"、"exit code 7"、"connection refused local dev"。
  デプロイの失敗には使いません — agents-deploy を使います。エラーのないパフォーマンス
  チューニングには使いません — agents-optimize を使います。VPC の設定には使いません —
  agents-build を使います。オブザーバビリティのセットアップやログの欠落には使いません —
  agents-optimize を使います。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# debug

AgentCore のエージェントや環境が正しく動作しない理由を診断します。

## 使うタイミング

- エージェントが誤った回答やエラーを返している
- ツール呼び出しが失敗している、またはタイムアウトしている
- エージェントはローカルでは動くが、デプロイ後に失敗する
- CloudWatch にログが表示されない
- AgentCore CLI が動作しない、または環境が壊れているように見える
- `agentcore` コマンドが見つからない、または前提条件が不足している

次の場合は使わないでください:

- デプロイの失敗（CDK のエラー、デプロイ中の IAM）→ `agents-deploy` を使います
- 新規プロジェクトのスキャフォールディング → `agents-get-started` を使います
- 品質の測定やモニタリングのセットアップ → `agents-optimize` を使います

## 入力

`$ARGUMENTS` は省略可能です。

```
/agents-debug                      # 対話形式 — 何が問題かを説明します
/agents-debug traces               # 最近のトレースを読んで説明します
/agents-debug logs                 # 最近のログからエラーを検索します
/agents-debug memory               # Memory の想起の問題に絞って診断します
/agents-debug doctor               # 環境の前提条件を確認します
```

## 手順

### ステップ 0: 問題の種類を判断する

開発者の問題が CLI 自体に関するもの（コマンドが見つからない、前提条件、環境のセットアップ）なら、[`references-ja/doctor.md`](references-ja/doctor.md) を読み込み、その診断チェックリストに従います。

問題がエージェントの動作に関するもの（誤った回答、エラー、タイムアウト、ツールの失敗）なら、以下のステップ 1 に進みます。

### ステップ 1: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。それより古い場合は、先に進む前に `agentcore update` を実行するよう、開発者に伝えてください。

### ステップ 2: 症状を理解する

質問します（またはコンテキストから推測します）。

> 「何が起きていますか?
>
> 1. エージェントがエラーメッセージを返す
> 2. エージェントが誤った、または役に立たない回答を返す
> 3. 特定のツール呼び出しが失敗している
> 4. Memory が動作しない（エージェントが物事を覚えていない）
> 5. エージェントが遅い、またはタイムアウトする
> 6. 特定のセッションで、エージェントが何をしたかを理解したい」

### ステップ 3: トレースとログを自動で読む

開発者にログの貼り付けを頼まず、直接読み取ります。

```bash
# 最近のトレースを一覧表示する
agentcore traces list --runtime <AgentName> --since 1h

# 最新のトレース ID を取得する
agentcore traces list --runtime <AgentName> --since 1h --limit 1

# トレースをダウンロードして読む
agentcore traces get <traceId> --runtime <AgentName>

# ログからエラーを検索する
agentcore logs --runtime <AgentName> --since 1h --level error

# ログから特定のパターンを検索する
agentcore logs --runtime <AgentName> --since 2h --query "timeout"
agentcore logs --runtime <AgentName> --since 2h --query "model access"
```

**重要:** CloudWatch の書き込みから読み取りまでのレイテンシーは、エンドツーエンドで**約 10 秒**です。これは、スパンが出力されてから、`agentcore traces get` や `agentcore run eval` で読み取れるようになるまでの遅延です。「トレースは取り込まれたが、評価の準備がまだできていない」という別個の期間は**ありません**。同じ取り込みのステップで、両方の経路が利用可能になります。古いスキルやドキュメントには、トレースが 30〜60 秒、評価が 2〜5 分と書かれていますが、どちらも古い情報です。エージェントを呼び出した直後なら、約 15 秒待てば、トレースの読み取りも評価も動作します。

エージェント名が指定されていない場合は、`agentcore/agentcore.json` を読んで取得します。

### ステップ 4: 症状別に診断する

---

## 症状: "model access denied" またはモデルのエラー

**最も多い原因:** モデルが、リージョンの Bedrock コンソールで有効になっていません。

対処:

1. AWS コンソール → Amazon Bedrock → Model access に移動する
2. エージェントが使うモデルを有効にする
3. アクセスが反映されるまで 1〜2 分待つ

**2 番目の原因:** 実行ロールに `bedrock:InvokeModel` がありません。

確認:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn $(agentcore status --json | jq -r '.runtimes[0].executionRoleArn') \
  --action-names bedrock:InvokeModel \
  --resource-arns "arn:aws:bedrock:*::foundation-model/*"
```

**3 番目の原因:** クロスリージョン推論プロファイルは、すべてのリージョンでモデルアクセスを必要とします。

地域のプレフィックスで始まるモデル ID は、その地域内でリクエストをルーティングする、クロスリージョン推論プロファイルです。

| プレフィックス | 地域 | 宛先リージョンの例 |
|---|---|---|
| `us.` | 米国 | us-east-1, us-east-2, us-west-2 |
| `eu.` | ヨーロッパ | eu-central-1, eu-west-1, eu-west-2, eu-west-3 |
| `apac.` | アジア太平洋 | ap-northeast-1, ap-southeast-1, ap-southeast-2, ap-south-1 |
| `global.` | 世界中のすべての商用リージョン | サポートされているすべてのリージョン |

AgentCore CLI は、デフォルトで `global.` を生成します（例: `global.anthropic.claude-sonnet-4-5-20250929-v1:0`）。どのプレフィックスでも、そのプロファイルがカバーするすべての宛先リージョンで、モデルアクセスを有効にする必要があります。`us.` のプロファイルでは米国のすべてのリージョン、`eu.` ではすべての EU リージョン、`global.` ではサポートされているすべてのリージョンで有効にします。すべてのモデルがすべてのプレフィックスをサポートしているわけではありません。`global.` は現在、一部のモデルでのみ利用できます。最大のスループットが欲しい場合は、利用可能であれば `global.` を、データレジデンシーの要件で推論の実行場所に制約がある場合は、地域のプレフィックスを使います。現在のモデルとプレフィックスの対応は、Bedrock の推論プロファイルのドキュメントで確認してください。

---

## 症状: ツール呼び出しが失敗する

**ステップ 1:** トレースから、失敗しているツール呼び出しを探します。

```bash
agentcore traces get <traceId> --runtime <AgentName>
```

エラーのステータスを持つ、ツール呼び出しのエントリを探します。

**ステップ 2:** Gateway の状態を確認します。

```bash
agentcore status --type gateway
agentcore fetch access --name <AgentName> --type agent
```

**ステップ 3:** ツール呼び出しの失敗でよくあるもの:

**Gateway の URL が設定されていない（ローカル開発）:**
環境変数 `AGENTCORE_GATEWAY_*_URL` は、デプロイ後にのみ設定されます。`agentcore dev` では、Gateway のツールは利用できません。これは想定どおりの動作で、エージェントはこれを適切に処理するべきです。

**ツール呼び出しでの認証の失敗:**

```bash
agentcore logs --runtime <AgentName> --since 1h --query "auth"
```

認証情報が正しく設定されていることを確認します: `agentcore status --type credential`

**Lambda 関数のエラー:**
Lambda 自体が失敗しています。Lambda のログを直接確認します。

```bash
aws logs tail /aws/lambda/<function-name> --since 1h
```

**ポリシーによる拒否:**
ポリシーエンジンがアタッチされている場合は、ポリシーの判定ログを確認します。

```bash
agentcore logs --runtime <AgentName> --since 1h --query "policy"
agentcore status --type policy-engine
```

---

## 症状: 誤った、または役に立たない回答

**ステップ 1:** トレースを読んで、エージェントの推論を確認します。

```bash
agentcore traces get <traceId> --runtime <AgentName>
```

トレースには、モデルの推論ステップ、行われたツール呼び出し、最終的なレスポンスが表示されます。次の点を確認します。

- エージェントは、正しいツールを使ったか
- ツール呼び出しは、期待どおりのデータを返したか
- システムプロンプトは、正しいコンテキストを提供しているか

**ステップ 2:** Memory が関係していないか確認します。
エージェントが Memory のコンテキストを使うべきなのに使っていない場合は、このスキルの後半にある「症状: Memory が動作しない」のセクションを参照するか、環境の問題であれば [`references-ja/doctor.md`](references-ja/doctor.md) を読み込みます。

**ステップ 3:** よくある原因:

- システムプロンプトが曖昧すぎる、または重要なコンテキストが欠けている
- エージェントが適切なツールを呼び出していない（ツールの説明を改善する必要がある）
- ツールが、予期しないデータ形式を返している
- タスクに対してモデル ID が適切でない（例: 複雑な推論に小さなモデルを使っている）

---

## 症状: Memory が動作しない

**Memory がセッションをまたいで永続化されない（LTM）:**

1. LTM の戦略（SEMANTIC または USER_PREFERENCE）が設定されていることを確認する:

```bash
agentcore status --type memory --json | jq '.memories[].strategies'
```

1. セッションが終了してから 5〜30 秒待つ — LTM の抽出は非同期です。事実が抽出される前に、エージェントはセッションを終了している必要があります。

2. セッション ID には UUID（v4）を使う — プラットフォームは最小 33 文字を要求します。"session-1" のような短い ID では、LTM が通知なしに失敗します。`agentcore invoke` は、要件を満たす ID をデフォルトで生成します。

3. Memory リソースが ACTIVE であることを確認する:

```bash
agentcore status --type memory
```

**セッション開始時に Memory が読み込まれない:**

1. 環境変数 `MEMORY_*_ID` が設定されていることを確認する:

```bash
agentcore status --type memory --json | jq '.memories[].id'
```

1. `actor_id` がセッション間で一貫していることを確認する — Memory はアクターごとにスコープされます。

2. 取得設定の名前空間のパスが、書き込み時に使った名前空間と一致していることを確認する。

---

## 症状: エージェントのタイムアウト

**ステップ 1:** トレースで、時間がどこで費やされているかを確認します。

```bash
agentcore traces get <traceId> --runtime <AgentName>
```

実行時間の長いステップ（モデル呼び出し、ツール呼び出し、Memory の操作）を探します。

**ステップ 2:** タイムアウトのよくある原因:

**エージェントの初期化が遅い:** アイドル後の最初の呼び出しだけが遅く、その後のリクエストは速い場合、エージェントが初期化に時間をかけすぎています。モジュールレベルの重いインポート、グローバルスコープでのデータベース接続、起動時の MCP クライアントの初期化を確認してください。コストの高いセットアップは、リクエストハンドラーに移すか、遅延初期化にします。最適化のガイダンスは、`agents-harden` スキルを参照してください。

**モデル呼び出しのタイムアウト:** モデルの応答に時間がかかりすぎています。時間が重要な処理には、より高速なモデルの使用を検討してください（たとえば、単純なタスクには Sonnet ではなく Haiku）。

**ツール呼び出しのタイムアウト:** Lambda や外部 API が遅いです。ツール自身のログを確認してください。

**Memory 取得のタイムアウト:** 大きな Memory ストアでは、セマンティック検索が遅くなることがあります。取得設定の `top_k` を減らすことを検討してください。

**VPC の接続の問題:** エージェントが VPC 内にある場合は、セキュリティグループのルールとルートテーブルを確認してください。VPC 固有のデバッグについては、`agents-build`（[`references-ja/vpc.md`](../agents-build/references-ja/vpc.md) を読み込む）を参照してください。

---

## 症状: `ServiceQuotaExceededException: maxVms limit exceeded`（観測される同時実行数が低いのに発生する）

CloudWatch の「同時セッション」メトリクスは控えめな数値（30〜50 程度）を示しているのに、`InvokeAgentRuntime` の呼び出しが `ServiceQuotaExceededException: maxVms limit exceeded` を返します。

**実際に起きていること:** CloudWatch の同時セッションのメトリクスは、稼働中の microVM の数と同じではありません。`maxVms` のクォータは、アカウントでアクティブなすべての環境を数えます。呼び出しは終わったものの、まだ回収されていない環境も含まれます。アイドルだがまだ回収されていない環境は、`idleRuntimeSessionTimeout`（デフォルトは 900 秒 / 15 分）が切れるか、明示的に停止するまで、クォータにカウントされ続けます。

コードがリクエストごとに新しいセッション ID を使い、`StopRuntimeSession` を呼び出していない場合、すべてのリクエストが、15 分間アイドル状態の環境を残し、クォータを消費します。

**対処の順序（クォータの引き上げを申請する前に、この順序で試す）:**

1. **各論理リクエストの完了後に、`StopRuntimeSession` を呼び出す。** このセッションでこれ以上リクエストを送らないなら、明示的に停止します。

   ```python
   client.stop_runtime_session(
       agentRuntimeArn=runtime_arn,
       runtimeSessionId=session_id,
   )
   ```

2. **関連するリクエスト間で、セッション ID を再利用する。** ユーザーの 1 つの操作が複数のバックエンド呼び出しを生む場合、呼び出しごとに新しいセッション ID を生成するのではなく、同じセッションにルーティングします。

3. **`idleRuntimeSessionTimeout` を下げる。** セッションが短命で、あらゆる場所に `StopRuntimeSession` を追加できない場合は、`agentcore/agentcore.json` のランタイムの `lifecycleConfiguration` を編集し、`agentcore deploy` を実行して、タイムアウトを下げます。

4. **上記を試してからにだけ、クォータの引き上げを申請する。** `agents-harden`（[`references-ja/limits.md`](../agents-harden/references-ja/limits.md) を読み込む）を参照してください。サポートチケットを直接起票するのではなく、Service Quotas コンソール（Amazon Bedrock AgentCore）から申請します。

完全なパターンは、`agents-harden` の「Session lifecycle management」セクションを参照してください。

---

## 症状: 呼び出しで 424 Failed Dependency が返る

これは通常、エージェントのコンテナが起動に失敗したか、初期化中にクラッシュしたことを意味します。

**ステップ 1:** エージェントのログで、起動時のエラーを確認します。

```bash
agentcore logs --runtime <AgentName> --since 30m --level error
```

**ステップ 2:** よくある原因:

**Python の依存関係が不足している:** エージェントのコードが、`pyproject.toml` にないパッケージをインポートしています。コンテナは起動しますが、最初のリクエストでクラッシュします。対処: 依存関係を追加して、再デプロイします。

**エントリーポイントのクラッシュ:** `main.py` が、インポート時または `app.run()` で例外を投げています。ログでトレースバックを確認してください。

**コンテナイメージの取得の失敗:** Container ビルドを使っている場合、ECR のイメージが存在しないか、実行ロールに `ecr:BatchGetImage` がない可能性があります。確認:

```bash
agentcore status --runtime <AgentName> --json
```

**Memory リソースが ACTIVE でない:** エージェントのコードが Memory を利用できると仮定しているのに、Memory リソースがまだ CREATING の状態だと、エントリーポイントが失敗することがあります。確認:

```bash
agentcore status --type memory
```

**初期化のタイムアウト:** エージェントが最初のリクエストを受け付ける準備ができるまでに時間がかかりすぎています。モジュールレベルでの重いインポート、同期的なデータベース接続、起動時の MCP クライアントの初期化が、サービスのヘルスチェックの時間枠を超えることがあります。症状は、最初の呼び出しだけ 424 になり、それ以降は正常に動作する、というものです。対処: コストの高いセットアップをモジュールレベルから外す、遅延初期化を使う、または本番トラフィックの前にエージェントをウォームアップします。パターンは、`agents-harden` の「Initialization time」セクションを参照してください。

---

## 症状: ローカルの呼び出しが connection-refused / exit code 7 で失敗する

通常はエージェントのバグではありません。開発サーバーが、想定とは別のポートで動作しています。

**`agentcore dev` がバインドするデフォルトのポート:**

| プロトコル | デフォルト |
|---|---|
| HTTP | 8080 |
| MCP | 8000 |
| A2A | 9000 |

**デフォルトのポートが使用中の場合**（2 つ目の開発セッション、前回の実行から残っているプロセス、8080 を使う別のサービスなど）、CLI は黙って**自動的にインクリメントします**: 8080 → 8081 → 8082。8080 にハードコードされたテストハーネスや `curl` のスクリプトは、エージェントが 8082 で正常に動作していても、`Connection refused`（curl の exit code 7）になります。

次の順に診断します。

1. `agentcore dev` が表示する CLI のバナーを読む — 実際にバインドされたポートと URL が表示されます。これが常に正しい情報源です。
2. バナーが消えている場合（ターミナルをクリアした、バックグラウンドで実行している）は、ログファイルを確認します。

   ```bash
   tail -20 agentcore/.cli/logs/dev/*.log
   ```

3. または、プロセスを直接探します。

   ```bash
   # macOS / Linux
   ps aux | grep -E 'agentcore dev|uvicorn' | grep -v grep
   lsof -iTCP -sTCP:LISTEN -n -P | grep -E '8080|8081|8082|8000|9000'
   ```

**対処の選択肢:**

- ポートを明示的に固定する: `agentcore dev --port 8080`
- デフォルトのポートを占有しているプロセスを終了する: `lsof -tiTCP:8080 -sTCP:LISTEN | xargs kill`
- テストハーネスにハードコードされたポートを、CLI の出力や環境変数から読み取るように更新する

これは、「ある日はローカルで動いたのに、翌日は失敗する」という報告の一般的な原因でもあります。実行の間にポートがずれたのです。

---

## 症状: Gateway のツール呼び出しが認証エラーで失敗する

**ステップ 1:** 認証の種類が、ターゲットの種類と一致していることを確認します。これは Gateway で最もよくあるエラーです — ターゲットに対して、誤ったアウトバウンド認証を使っています。

| ターゲットの種類 | 有効なアウトバウンド認証 |
|---|---|
| `mcp-server` | `none`、`oauth`、または IAM（API 経由の SigV4） |
| `lambda-function-arn` | IAM のみ（自動） |
| `open-api-schema` | `oauth` または `api-key`（必須） |
| `api-gateway` | `none`、`api-key`、または IAM |
| `smithy-model` | IAM または `oauth` |

**ステップ 2:** OAuth トークンの期限切れを確認します。Gateway ターゲットが OAuth を使う場合、アクセストークンが期限切れになっている可能性があります。認証に関連するエラーを探します。

```bash
agentcore logs --runtime <AgentName> --since 1h --query "auth"
agentcore logs --runtime <AgentName> --since 1h --query "401"
agentcore logs --runtime <AgentName> --since 1h --query "403"
```

トークンが期限切れになっている場合は、OAuth の認証情報プロバイダーのトークンエンドポイントに到達できること、クライアントクレデンシャルがまだ有効であることを確認します。OAuth を使う MCP サーバーターゲットでは、Gateway がトークンの更新を自動で処理します。それが失敗している場合は、認証情報プロバイダーの設定が誤っている可能性があります。

**ステップ 3:** 認証情報が設定されていることを確認します。

```bash
agentcore status --type credential
agentcore status --type gateway --json
```

---

## 症状: トレースが表示されない

**約 15 秒待ってください** — 呼び出しからトレースが利用可能になるまでには、短い遅延（通常は約 10 秒）があります。

約 30 秒たってもトレースが表示されない場合:

1. エージェントのデプロイ時に、オブザーバビリティが有効だったことを確認する
2. エージェントが実際に呼び出されたことを確認する: `agentcore logs --runtime <AgentName> --since 1h`
3. 実行ロールの CloudWatch 権限を確認する

---

## 症状: CloudWatch のログが表示されない

これは、特に Container/Docker ビルドで、最もよくあるオブザーバビリティの問題です。

AgentCore は、生の stdout を取得しません。OpenTelemetry を使って、ログを CloudWatch に送ります。次の 3 つが満たされている必要があります。

**1. エントリーポイントを `opentelemetry-instrument` でラップする。**

CodeZip ビルドでは、これが自動で行われます。Docker/Container ビルドでは手動で追加する必要があります — これが、見落とされやすい第 1 の点です。

Dockerfile の CMD で:

```dockerfile
# ✅ 正しい — opentelemetry-instrument でラップされている
CMD ["opentelemetry-instrument", "python", "main.py"]

# ❌ 誤り — OTEL のラッパーがなく、ログが表示されない
CMD ["python", "main.py"]
```

**2. ランタイムの IAM ロールに、CloudWatch と X-Ray の権限が必要。**

```
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents    → /aws/bedrock-agentcore/runtimes/* に絞り込む
xray:PutTelemetryRecords
xray:PutTraceSegments → * に絞り込む
```

CodeZip で AgentCore CLI を使っている場合、CDK のスキャフォールドがこれらを自動で追加します。カスタムのロールや Container ビルドを使っている場合は、存在することを確認してください。

**3. `print()` ではなく、Python の `logging` モジュールを使う。**

OTEL は `logging` に自動でフックします — カスタムのハンドラーは不要です。`print()` の出力は CloudWatch に表示されません。

```python
import logging
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# ✅ これは CloudWatch に表示される
logger.info("Processing request")

# ❌ これは CloudWatch に表示されない
print("Processing request")
```

**これも確認してください:** CloudWatch Transaction Search がアカウントで有効になっていること。これが無効だと、トレースとスパンが GenAI Observability のダッシュボードに表示されません。

### Terraform/CDK/IaC でデプロイしたランタイムでログが欠ける

よくあるパターン: Terraform、CDK、またはカスタムの IAM ロールでデプロイしたランタイムが正しく動作して（レスポンスを返して）いるのに、CloudWatch のログストリームが表示されません。同じエージェントのコードを AgentCore コンソールからデプロイすると、ログは正常に記録されます。

これはほぼ間違いなく、IAM のスコープの問題です。コンソールからデプロイしたランタイムの実行ロールには、デフォルトで広範な CloudWatch 権限が付与されます。IaC のテンプレートでは、これらの権限を `/aws/bedrock-agentcore/runtimes/*` に狭く絞り込むことが多く、それがログストリームの作成を妨げます。

**対処:** `logs:DescribeLogGroups` には、絞り込んだリソースではなく、`Resource: "*"` が必要です。他の logs のアクションは、ランタイムのロググループに絞り込めます。

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:DescribeLogGroups"
  ],
  "Resource": "*"
},
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "arn:aws:logs:<REGION>:<ACCOUNT_ID>:log-group:/aws/bedrock-agentcore/runtimes/*:*"
}
```

実行ロールの IAM ポリシーを更新したら、`agentcore deploy` でランタイムを再デプロイして、新しい権限を反映させます。

---

## 症状: ストリーミング接続がレスポンスの途中で切れる

エージェントが SSE やロングポーリングのレスポンスを使っていて、ストリームの途中で接続が切れます。クライアントコードでの症状:

- `RemoteProtocolError: peer closed connection without sending complete message body`
- ストリームのイテレーション中の `IncompleteRead` 例外
- 通知のない切断 — エラーも `[DONE]` イベントもなく、レスポンスがただ止まる
- 複数のツールを使う会話（5 回以上連続してツールを呼び出す）で発生する
- クライアント側のタイムアウトよりもずっと前に失敗する

**根本原因:** ストリーミング接続に対する、インフラ層のアイドルタイムアウトです。レスポンスのストリーム上でデータが数分間流れない場合（たとえば、ツールの実行中の無音の期間）、ランタイムの前段にあるロードバランサーが TCP 接続を終了します。

タイムアウトの対象は、リクエスト全体の所要時間ではなく、**ストリームを流れるデータ**です。定期的にバイトを出力している限り、接続は維持されます。

**対処: 長時間実行されるツールの実行中に、キープアライブのイベントを送出する。**

ストリーミングのエントリーポイントの Python パターン:

```python
import asyncio
import json
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

async def emit_keepalive(tool_task):
    """tool_task の実行中、30 秒ごとにハートビートのイベントを yield する。"""
    while not tool_task.done():
        yield f"data: {json.dumps({'type': 'heartbeat'})}\n\n"
        try:
            await asyncio.wait_for(asyncio.shield(tool_task), timeout=30)
        except asyncio.TimeoutError:
            continue  # ツールはまだ実行中。もう一度ハートビートを送出する

@app.entrypoint
async def invoke(payload, context):
    async def stream():
        tool_task = asyncio.create_task(run_long_tool(payload))

        # ツールの実行中、ハートビートを送出する
        async for event in emit_keepalive(tool_task):
            yield event

        # ツールが完了 — 実際の結果を送出する
        result = await tool_task
        yield f"data: {json.dumps({'type': 'result', 'content': result})}\n\n"
        yield "data: [DONE]\n\n"

    return stream()
```

ハートビートの間隔は約 30 秒にしてください。長すぎるとアイドルタイムアウトに達するおそれがあり、短すぎると帯域を無駄にします。

**クライアント側では、バイトをユーザーに見せる前に、ハートビートのイベントをフィルタリングします。**

```python
for chunk in response.iter_lines():
    if not chunk:
        continue
    data = json.loads(chunk.removeprefix(b"data: "))
    if data.get("type") == "heartbeat":
        continue  # キープアライブは無視する
    # 実際のイベントを処理する
```

**代替手段: 投げっぱなしのパターンには、SDK の非同期タスク API を使う。** クライアントが結果を待つ必要がない場合は、`add_async_task` / `complete_async_task` で作業を登録し、呼び出しをすぐに返します。`agents-harden` の「Long-running background tasks」セクションを参照してください。

---

## 症状: 同時に実行したエージェントの呼び出しのトレースが統合されて表示される

一意の `runtimeSessionId` を持つ複数のエージェントの呼び出しを並列に実行しているのに、AI Observability のダッシュボードでは、それらが 1 つのセッションとしてグループ化され、1 回の実行を切り分けられなくなります。データプレーンのログでは、セッション ID が正しくリクエスト ID と 1 対 1 で一意になっていますが、トレースのビューでは統合されたままです。

**最も多い原因: 呼び出し元が Active Tracing を有効にしていない**ため、上流のスパンが `Sampled=0` で届いています。AgentCore は、デフォルトで上流のトレースサンプリングの判断を尊重します。親のコンテキストが「サンプリングしない」と言っていれば、スパンは破棄され、同時に実行した呼び出しがダッシュボードで統合されて見えることがあります。

**呼び出し元の種類ごとの対処:**

**Lambda の呼び出し元:** Lambda 関数で Active Tracing を有効にします。

```bash
aws lambda update-function-configuration \
  --function-name my-caller-function \
  --tracing-config Mode=Active
```

または Lambda コンソールで: Configuration → Monitoring and operations tools → AWS X-Ray → Active tracing。

**ECS / EC2 / コンテナの呼び出し元:** AWS X-Ray SDK を初期化し、AgentCore へのアウトバウンド呼び出しが計装されるようにします。Python では、`aws-xray-sdk` を使い、SDK にパッチを当てます。

```python
from aws_xray_sdk.core import xray_recorder, patch_all
patch_all()  # boto3、requests などにパッチを当てる
```

**X-Ray を使わない SDK の直接の呼び出し元:** 上流のトレースを有効にできない場合は、エージェントに環境変数を設定して、ランタイムに強制的にサンプリングさせます。

```
OTEL_TRACES_SAMPLER=always_on
```

これにより、親のコンテキストのサンプリングの判断に関係なく、ランタイムがすべてのトレースをサンプリングします。トレードオフとして、トレースのコストは高くなりますが、トレースは正しくなります。

### これも確認: エージェントの ARN ではなく、エンドポイントの ARN で呼び出している

トレースに、子スパンのない最上位の `AgentCore.Runtime.Invoke` スパンだけが表示される場合は、呼び出し元が使っている ARN を確認してください。呼び出し先は、エージェントランタイムの ARN であるべきです。

```
arn:aws:bedrock-agentcore:<region>:<account>:runtime/<runtime-name>
```

エンドポイントの ARN ではありません。

```
arn:aws:bedrock-agentcore:<region>:<account>:runtime/<runtime-name>/runtime-endpoint/DEFAULT
```

エンドポイントの ARN で呼び出すと、完全なトレースの計装の経路を迂回することがあります。これは気づきにくい落とし穴です。どちらの ARN でも成功するレスポンスが返りますが、完全なトレースが生成されるのは、エージェントの ARN の場合だけです。

---

## 症状: ランタイムが数時間 DELETING のまま止まる

`DeleteAgentRuntime` を呼び出し、`status: DELETING` の成功レスポンスを受け取りましたが、ランタイムがその状態のまま 30 分以上止まっています。デフォルトのエンドポイントを別途削除しようとすると、`ConflictException: Default endpoints are removed when you delete the agent.` が返ります。

**起きていること:** 削除のワークフローが、サービス側で止まっています。`DeleteAgentRuntime` をリトライしても効果はありません。呼び出し自体は即座に成功し（DELETING を返します）、止まっているのはバックエンドのワークフローです。利用者側のツールでは、強制的に完了させられません。

**対応:**

1. **リトライし続けない。** ワークフローの停止は解消されません。
2. **AWS サポートにケースを起票する。** https://console.aws.amazon.com/support から。次の情報を含めてください。
   - AWS アカウント ID
   - リージョン
   - ランタイムの ARN（または `agentRuntimeId`）
   - 元の `DeleteAgentRuntime` 呼び出しの `requestId` とタイムスタンプ（CloudTrail から）
   - ランタイムが DELETING の状態になってからの経過時間
3. **その間は回避策で凌ぐ。** 出荷を続ける必要がある場合は、別の名前で新しいランタイムをデプロイします。止まったリソースに、作業を妨げさせないでください。

削除が止まったことで残った孤立したリソース（ENI、ワークロードアイデンティティ）は、同じケースの中で、サービスチームによる手動のクリーンアップが必要になる場合があります。

---

## フレームワーク固有の問題

**LangGraph — モデルの形式:**
古いバージョンの `langchain-aws` は、クロスリージョンのプレフィックスなしのモデル ID を必要としていました。最近のバージョンは、クロスリージョン推論プロファイルをサポートしている可能性があります。インストールされているバージョンを確認してください。

```bash
pip show langchain-aws | grep Version
```

LangGraph でモデルのエラーが発生する場合は、プレフィックスなしの ID を試してください。

```python
# お使いの langchain-aws のバージョンで、クロスリージョンのプレフィックスがエラーになる場合:
llm = init_chat_model("anthropic.claude-sonnet-4-5-20250929-v1:0", model_provider="bedrock_converse")

# お使いのバージョンがクロスリージョンプロファイルをサポートしている場合（us. = 米国、eu. = ヨーロッパ、apac. = アジア太平洋、global. = 世界中）:
llm = init_chat_model("global.anthropic.claude-sonnet-4-5-20250929-v1:0", ...)
```

最新の langchain-aws のリリースノートで確認してください: https://github.com/langchain-ai/langchain-aws/releases — クロスリージョン推論プロファイルのサポートは進化しています。

**Google ADK — Gemini のみ:**
ADK は Gemini のモデルでしか動作しません。ADK でモデルのエラーが出る場合は、`GEMINI_API_KEY` が設定されていること、`gemini-*` のモデル ID を使っていることを確認してください。

**A2A エージェント — ポートの誤り:**
A2A サーバーは、ポート 9000 で動作する必要があります。A2A のエージェントが応答しない場合は、誤って 8080 で動作していないか確認してください。

---

## トレースの読み方

トレースは、エージェントの 1 回の呼び出しの完全な実行経路を示します。主なセクション:

- **モデルの呼び出し** — モデルに何が尋ねられ、何を応答したか
- **ツール呼び出し** — どのツールが、どのような入力で呼び出され、何を返したか
- **Memory の操作** — Memory から何を読み取り、何を書き込んだか
- **ポリシーの判定** — 何が許可または拒否されたか（ポリシーエンジンがアタッチされている場合）
- **レイテンシーの内訳** — 各コンポーネントで費やされた時間

```bash
# 詳細に調べるために、トレースをファイルにダウンロードする
agentcore traces get <traceId> --runtime <AgentName> --output trace.json
cat trace.json | jq '.trace.orchestrationTrace.modelInvocationOutput'
```

## 出力

- 根本原因を含む、特定の失敗の診断
- 具体的な修正コマンドやコードの変更
- トレースを読んでいる場合は、トレースが示していることの説明
- 修正が debug の範囲外にある場合は、適切なスキルへの引き継ぎ

## 診断後 — 引き継ぎ

根本原因を特定したら、修正を担当するスキルに引き継ぎます。

| 根本原因 | 引き継ぎ先 | 詳細 |
|---|---|---|
| Memory の設定ミス（戦略、名前空間、組み込みの誤り） | `agents-build` | [`references-ja/memory.md`](../agents-build/references-ja/memory.md) を読み込む |
| アプリからのエージェント呼び出しが動作しない（認証、URL、ストリーミング） | `agents-build` | [`references-ja/integrate.md`](../agents-build/references-ja/integrate.md) を読み込む |
| VPC の接続（RDS に到達できない、インターネットがない、AZ のエラー） | `agents-build` | [`references-ja/vpc.md`](../agents-build/references-ja/vpc.md) を読み込む |
| マルチエージェントの委任が動作しない | `agents-build` | [`references-ja/multi-agent.md`](../agents-build/references-ja/multi-agent.md) を読み込む |
| カスタムのリクエストヘッダーがエージェントのコードに届かない | `agents-build` | [`references-ja/request-headers.md`](../agents-build/references-ja/request-headers.md) を読み込む |
| 別のアカウントのアプリからのクロスアカウント呼び出し | `agents-build` | [`references-ja/integrate.md`](../agents-build/references-ja/integrate.md) を読み込む（クロスアカウントのセクション） |
| Gateway の認証の設定ミス（401、認証の種類の誤り） | `agents-connect` | Gateway の認証マトリクス |
| Gateway のターゲットの種類に関する質問（Lambda、OpenAPI、MCP、API Gateway のどれか） | `agents-connect` | 「Gateway とは何か — そして何ではないか」のセクション |
| ポリシーが予期せず拒否している（Cedar、ツールでのアクセス拒否） | `agents-connect` | [`references-ja/policy.md`](../agents-connect/references-ja/policy.md) を読み込む |
| オブザーバビリティが未設定（ログもトレースも表示されない） | `agents-optimize` | [`references-ja/observability.md`](../agents-optimize/references-ja/observability.md) を読み込む |
| コールドスタート / 初期化が遅すぎる | `agents-harden` | 「Initialization time」のセクション |
| セッションのライフサイクル / `maxVms` / `StopRuntimeSession` | `agents-harden` | 「Session lifecycle management」のセクション |
| 長時間実行のバックグラウンドタスクが回収される | `agents-harden` | 「Long-running background tasks」のセクション |
| JWT によるインバウンド認証の失敗（403、`allowedClients`/`allowedAudience`、発行者の不一致） | `agents-harden` | 「Inbound auth」のセクション |
| スロットリング / クォータのエラー / 上限引き上げの申請 | `agents-harden` | [`references-ja/limits.md`](../agents-harden/references-ja/limits.md) を読み込む |
| デプロイしたアーティファクトが古い、またはバージョンが違う | `agents-deploy` | 再デプロイのワークフロー |
| 環境の破損（CLI、認証情報、Node、uv） | [`references-ja/doctor.md`](references-ja/doctor.md) を読み込む | このスキルの中で完結する |

診断を明確に伝えたうえで、次にどのスキルを使うかを開発者に伝えてください。同じセッションで、参照先のスキルを読み込める場合は、そうしてください。
