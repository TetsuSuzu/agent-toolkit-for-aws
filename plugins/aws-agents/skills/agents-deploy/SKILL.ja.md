---
name: agents-deploy
description: >
  エージェントを AWS にデプロイするとき、またはデプロイが失敗したときに使います。
  事前検証、CDK/IAM/クォータのエラー診断、バージョン管理、ロールバック、
  カナリアデプロイを扱います。トリガー: "deploy my
  agent"、"agentcore deploy"、"deploy failed"、"CDK error"、"rollback"、
  "canary deploy"、"pin version"、"redeploy"、"deploy stuck"。
  本番向けの堅牢化には使いません — agents-harden を使います。デプロイ前の機能追加には
  使いません — agents-build または agents-connect を使います。
  VPC の設定エラーには使いません — agents-build を使います。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# deploy

AgentCore のエージェントを AWS にデプロイします。またはデプロイが失敗した理由を診断します。

## 使うタイミング

- デプロイの準備ができており、先に設定を検証したい
- `agentcore deploy` がエラーで失敗した
- 実際にデプロイせずに、デプロイで何が作成されるかをプレビューしたい
- 特定のターゲット（ステージング、本番）にデプロイしたい
- 以前のバージョンへのロールバック、特定のバージョンへの固定、カナリアデプロイの設定が必要

## 入力

`$ARGUMENTS` は省略可能です。

```
/agents-deploy                     # 対話形式 — 事前チェック、または失敗の診断
/agents-deploy preflight           # デプロイ前に、設定と IAM を検証します
/agents-deploy diagnose            # 失敗したデプロイを診断します（エラーを貼り付けるか、ログを読みます）
/agents-deploy preview             # デプロイせずに、何が作成されるかを表示します
/agents-deploy rollback            # 以前のバージョンにロールバックします
```

## 手順

### ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。それより古い場合は、先に進む前に `agentcore update` を実行するよう、開発者に伝えてください。

### ステップ 1: 状況を判断する

`agentcore/agentcore.json` と、存在すれば `agentcore/aws-targets.json` を読みます。

質問します（またはコンテキストから推測します）。

> 「次のどれですか。
>
> 1. これからデプロイするので、先にすべてを確認したい
> 2. 失敗したデプロイに対処している — どのようなエラーが出ましたか?
> 3. ロールバックや、特定のバージョンへの固定が必要」

開発者に、バージョン管理、ロールバック、カナリアデプロイが必要な場合は、[`references-ja/versioning.md`](references-ja/versioning.md) を読み込み、その指示に従います。

---

## Path A: 事前検証

`agentcore deploy` の前に、次のチェックを実行します。

### チェック 1: 設定ファイルを検証する

開発者に、次のコマンドを実行するよう示します。

```bash
agentcore validate
```

これにより、CDK が開始される前に、不正な形式の `agentcore.json` を検出できます。

### チェック 2: リージョンの整合性を確認する

デプロイの失敗で最も多いのは、リージョンの不一致です。開発者に、確認用の次のコマンドを示します。

```bash
# 設定されている AWS のリージョン
aws configure get region

# デプロイターゲットのリージョン
cat agentcore/aws-targets.json

# 実際に認証されているアカウント
aws sts get-caller-identity
```

`aws-targets.json` の `region` は、`aws configure` のデフォルトのリージョンと一致している必要があります。`account` は、`sts get-caller-identity` のアカウント ID と一致している必要があります。

### チェック 3: Bedrock のモデルアクセスを確認する

開発者に、リージョンで有効になっているモデルを確認する、次のコマンドを示します。

```bash
aws bedrock list-foundation-models --region $(aws configure get region) \
  --query 'modelSummaries[?modelLifecycle.status==`ACTIVE`].modelId' \
  --output table
```

クロスリージョン推論プロファイルの ID は、推論の実行場所を制御する地域のプレフィックス（`us.`、`eu.`、`apac.`）または `global.` を使います。CLI はデフォルトで `global.` を生成し（例: `global.anthropic.claude-sonnet-4-5-20250929-v1:0`）、任意の商用リージョンにルーティングされます。地域のプレフィックスは、推論をその地域内にとどめます（たとえば、`eu.` は EU のリージョンにとどまります）。どのプレフィックスでも、そのプロファイルがカバーするすべての宛先リージョンで、モデルアクセスを有効にする必要があります。各プレフィックスにどのリージョンが含まれるかは、Bedrock のドキュメントで確認してください。

### チェック 4: デプロイされる内容をプレビューする

```bash
agentcore deploy --dry-run
agentcore deploy --diff
```

`--dry-run` は、作成されるリソースを表示します。`--diff` は、現在デプロイされているものとの CDK の差分を表示します。

### チェック 5: IAM 権限を確認する

開発者に、必要な権限と、次の確認用のコマンドを示します。

```bash
aws iam simulate-principal-policy \
  --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
  --action-names iam:CreateRole \
  --resource-arns "arn:aws:iam::*:role/*BedrockAgentCore*"
```

### デプロイを実行する

```bash
agentcore deploy -y          # 自動で確認する（エイリアス: agentcore dp -y）
agentcore deploy -y -v       # 詳細表示 — リソースレベルのイベントを表示する
agentcore deploy --target staging -y   # 特定のターゲットにデプロイする
```

**Memory のプロビジョニングに関する注意:** プロジェクトに Memory が含まれる場合、Memory リソースが ACTIVE になるまで、デプロイに 2〜5 分余計にかかります。これは正常で、エラーではありません。状態を確認します。

```bash
agentcore status --type memory
```

---

## Path B: 失敗したデプロイを診断する

### ステップ B1: エラーを読む

開発者がエラーを貼り付けた場合は、それを直接診断します。そうでなければ、デプロイのログを読みます。

```bash
# 最近のデプロイログを表示する
ls -lt agentcore/.cli/logs/
cat agentcore/.cli/logs/deploy-*.log 2>/dev/null | tail -100
```

### ステップ B2: 既知の失敗パターンと照合する

**IAM 権限のエラー:**

```
User: arn:aws:iam::123456789012:user/dev is not authorized to perform: iam:CreateRole
```

対処: 必要な IAM 権限を付与します（上のチェック 5 を参照）。デプロイを行う ID には、`*BedrockAgentCore*` のロールに絞り込んだ IAM の書き込みアクセスが必要です。

**CDK の bootstrap が未実行:**

```
This stack uses assets, so the toolkit stack must be deployed to the environment
```

対処:

```bash
npx cdk bootstrap aws://<YOUR_ACCOUNT_ID>/<REGION>
```

**ECR の認可エラー:**

```
no basic auth credentials
Error response from daemon: Head "https://<YOUR_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/..."
```

対処:

```bash
aws ecr get-login-password --region <REGION> | \
  docker login --username AWS --password-stdin <YOUR_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
```

**デプロイ中のモデルアクセス拒否:**

```
ValidationException: The provided model identifier is invalid
```

対処: Bedrock コンソール → Model access で、モデルを有効にします。`agentcore.json` のモデル ID が、ターゲットのリージョンで有効なモデルと一致していることを確認します。

**リージョンの不一致:**

```
Stack ... is in region us-east-1 but the target is us-west-2
```

対処: `agentcore/aws-targets.json` を、`aws configure` のデフォルトのリージョンに合わせて更新するか、`aws configure set region <REGION>` を実行します。

**Memory が CREATING のまま止まる:**

```
Memory resource is in CREATING state after 10 minutes
```

これは珍しいケースです。通常のプロビジョニングは 2〜5 分かかります。確認:

```bash
agentcore status --type memory --json
```

止まっている場合は、Memory リソースを削除して、もう一度追加してみてください。

**サービスクォータの超過:**

```
LimitExceededException: Account limit for AgentCore runtimes exceeded
```

対処: AWS コンソール → Service Quotas → Amazon Bedrock AgentCore で、クォータの引き上げを申請します。

### ステップ B3: 修正後に再実行する

```bash
agentcore deploy -y
```

同じエラーが再発する場合は、`agentcore status` で、すべてのリソースの現在の状態を確認します。

```bash
agentcore status
agentcore status --state pending-removal  # 削除対象としてマークされたリソース
```

---

## 複数のターゲットにデプロイする

`agentcore/aws-targets.json` にターゲットを定義します。

```json
[
  {
    "name": "staging",
    "description": "Staging environment",
    "account": "123456789012",
    "region": "us-east-1"
  },
  {
    "name": "production",
    "description": "Production environment",
    "account": "987654321098",
    "region": "us-west-2"
  }
]
```

特定のターゲットにデプロイします。

```bash
agentcore deploy --target staging -y
agentcore deploy --target production -y
```

## 出力

- 事前チェックの結果と、見つかった問題に対する具体的な修正
- デプロイの失敗の診断と、具体的な修正
- 修正の適用後に実行する、デプロイのコマンド
