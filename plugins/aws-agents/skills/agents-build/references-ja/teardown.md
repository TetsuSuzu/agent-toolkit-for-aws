# teardown

プロジェクトから個別のリソースを削除するか、デプロイ全体を撤去します。

## 使うタイミング

- プロジェクトから、Gateway、Memory、認証情報、Evaluator などのリソースを削除したい
- デプロイ済みのエージェントを削除し、すべての AWS リソースをクリーンアップしたい
- サンドボックスアカウントで試行錯誤しており、最初からやり直したい
- 動かなくなった、または不要になったリソースを削除する必要がある

## 手順

### プロジェクトから個別のリソースを削除する

`agentcore remove` を使って、`agentcore.json` からリソースを削除します。これはリソースを削除対象としてマークするもので、実際の AWS リソースは次回の `agentcore deploy` で削除されます。

```bash
# Memory リソースを削除する
agentcore remove memory --name MyMemory

# Gateway ターゲットを削除する
agentcore remove gateway-target --name WeatherTools --gateway MyGateway

# Gateway を削除する（先にすべてのターゲットを削除してください）
agentcore remove gateway --name MyGateway

# 認証情報を削除する
agentcore remove credential --name MyAPIKey

# Evaluator を削除する
agentcore remove evaluator --name ResponseQuality

# オンライン評価の設定を削除する
agentcore remove online-eval --name production_monitor

# ポリシーを削除する
agentcore remove policy --name SpendingLimit --engine MyPolicyEngine

# ポリシーエンジンを削除する（先にすべてのポリシーを削除してください）
agentcore remove policy-engine --name MyPolicyEngine
```

削除したら、デプロイして変更を適用します。

```bash
agentcore deploy -y
```

デプロイする前に、削除待ちのものを確認します。

```bash
agentcore status --state pending-removal
```

### マルチエージェントのプロジェクトからエージェントを削除する

プロジェクトに複数のエージェント（ランタイム）がある場合は、そのうちの 1 つを削除できます。

```bash
agentcore remove agent --name SecondAgent
agentcore deploy -y
```

これにより、そのエージェントのランタイム、エンドポイント、関連リソースが AWS から削除されます。`app/<AgentName>/` にあるエージェントのコードは削除されません。不要であれば手動で削除してください。

### デプロイ全体を撤去する

プロジェクトでデプロイしたすべての AWS リソースを削除するには:

```bash
# 破棄されるものをプレビューする
agentcore deploy --diff

# すべてのリソースを破棄する
npx cdk destroy --app "npx ts-node agentcore/cdk/bin/cdk.ts" --force
```

別の方法として、CloudFormation スタックを直接削除することもできます。

```bash
# スタック名を調べる
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, '<ProjectName>')].StackName"

# 削除する
aws cloudformation delete-stack --stack-name <StackName>

# 削除が完了するまで待つ
aws cloudformation wait stack-delete-complete --stack-name <StackName>
```

### 削除されるものと、されないもの

| リソース | `cdk destroy` で削除 | 備考 |
|---|---|---|
| AgentCore Runtime | ✅ | すべてのエンドポイントとバージョンを含む |
| Memory リソース | ✅ | Memory のデータは完全に削除される |
| Gateway とターゲット | ✅ | |
| 認証情報 | ✅ | Secrets Manager のエントリも削除される |
| ポリシーエンジンとポリシー | ✅ | |
| Evaluator の定義 | ✅ | |
| オンライン評価の設定 | ✅ | |
| IAM ロール | ✅ | CDK が作成したもの |
| CloudWatch のロググループ | ❌ | 削除後も残る — 必要に応じて手動で削除する |
| ECR イメージ（コンテナビルド） | ❌ | 残る — リポジトリを手動で削除する |
| CDK bootstrap スタック | ❌ | プロジェクト間で共有される — CDK を一切使わなくなるまで削除しない |
| ローカルのプロジェクトファイル | ❌ | `agentcore/`、`app/` — 手動で削除する |

### CloudWatch のロググループをクリーンアップする

ロググループは、スタックを削除したあとも残ります。クリーンアップするには:

```bash
# AgentCore のロググループを一覧表示する
aws logs describe-log-groups \
  --log-group-name-prefix /aws/bedrock-agentcore/ \
  --query "logGroups[].logGroupName"

# 特定のロググループを削除する
aws logs delete-log-group --log-group-name /aws/bedrock-agentcore/runtimes/<AGENT_ID>-DEFAULT
```

### ECR リポジトリをクリーンアップする（コンテナビルド）

```bash
# AgentCore の ECR リポジトリを一覧表示する
aws ecr describe-repositories \
  --query "repositories[?contains(repositoryName, 'bedrock-agentcore')].repositoryName"

# リポジトリを、すべてのイメージごと削除する
aws ecr delete-repository --repository-name <repo-name> --force
```

### 動かなくなったリソースへの対処

ランタイムが 30 分以上 DELETING 状態のまま止まっている場合は、`agents-debug` の "Runtime stuck in DELETING" セクションを参照してください。要点は、何度もリトライせずに、ランタイムの ARN と、CloudTrail にある元の削除リクエスト ID を添えて、AWS サポートにケースを起票することです。

## よくある問題

**"Can't remove gateway — targets still attached"**
先にすべての Gateway ターゲットを削除してから、Gateway を削除します。

```bash
agentcore remove gateway-target --name Target1 --gateway MyGateway
agentcore remove gateway-target --name Target2 --gateway MyGateway
agentcore remove gateway --name MyGateway
```

**"Can't remove policy engine — policies still attached"**
先にすべてのポリシーを削除してから、エンジンを削除します。

```bash
agentcore remove policy --name Policy1 --engine MyEngine
agentcore remove policy-engine --name MyEngine
```

**"Resource shows pending-removal but deploy doesn't delete it"**
`agentcore status --state pending-removal` を実行し、そのリソースが一覧に出ているか確認してください。削除されないままデプロイが完了した場合は、CDK の出力でエラーを確認してください。依存関係が原因で、削除が通知なしに失敗した可能性があります。

## 出力

- 対象のリソースを削除するための CLI コマンド
- 削除後に残るものと、そのクリーンアップ方法についてのガイダンス
- 元に戻せないデータ損失（Memory のデータ、認証情報）に関する警告
