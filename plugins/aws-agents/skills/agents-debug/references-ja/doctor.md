# doctor

環境を確認し、AgentCore CLI を使うために何が必要かを正確に伝えます。

## 使うタイミング

- `agentcore` コマンドが見つからない、または CLI が正しく動作しない
- `agentcore create` や `agentcore deploy` が、環境のエラーで即座に失敗する
- 開発者が、環境が正しく設定されているか確信が持てない
- OS やツールの更新後に、以前は動いていたものが動かなくなった

次の場合は使わないでください:

- 新規プロジェクトの作成や、使い始め → `agents-get-started` を使います
- 環境に関係しないデプロイの失敗（CDK のエラー、IAM）→ `agents-deploy` を使います
- エージェントのランタイムのエラー → `agents-debug` を使います

## 入力

引数は不要です。

## 手順

各チェックを実行して、結果を報告します。不足しているものについては、正確な修正コマンドを示してください。「X をインストールしてください」と言うだけでは不十分です。

### チェック 1: AgentCore CLI

```bash
agentcore --version
```

**バージョンが返らずにコマンドがエラーになる場合:**

`which agentcore` を実行して、何がインストールされているか確認します。

- パスが `/usr/local/lib/python*/site-packages/` などの Python の場所にある → 古い Starter Toolkit が、新しい CLI を隠しています。アンインストールしてください（下記を参照）。
- パスが Node.js ベースの場所にあるのに、エラーになる → Node.js のバージョンが誤っている可能性があります。チェック 2 に進みます。
- パスが返らない → CLI がインストールされていません。

**見つからない場合:**

```bash
npm install -g @aws/agentcore
```

Node.js 20 以上が必要です。`npm` がない場合は、先に Node.js をインストールしてください: https://nodejs.org

**古い Starter Toolkit がインストールされている場合**（Python ベースの `agentcore` コマンド）:

```bash
# 先に、古い CLI をアンインストールする
pip uninstall bedrock-agentcore-starter-toolkit
# または: pipx uninstall bedrock-agentcore-starter-toolkit
# または: uv tool uninstall bedrock-agentcore-starter-toolkit

# その後、新しい CLI をインストールする
npm install -g @aws/agentcore
```

### チェック 2: Node.js のバージョン

```bash
node --version
```

Node.js 20.x 以降が必要です。それより古い場合:

- macOS: `brew install node`、または https://nodejs.org からダウンロードする
- Linux: `nvm install 20` を使う（https://github.com/nvm-sh/nvm）

### チェック 3: uv（Python のパッケージマネージャー）

```bash
uv --version
```

`uv` は、エージェントのコード用の Python 仮想環境を管理します。Python のエージェントには必須です。

**見つからない場合:**

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# または pip 経由
pip install uv

# または Homebrew 経由
brew install uv
```

インストール後は、ターミナルを再起動するか、`source ~/.bashrc`（または `~/.zshrc`）を実行します。

### チェック 4: AWS の認証情報

```bash
aws sts get-caller-identity
```

**AWS CLI が見つからない場合:**

```bash
# macOS
brew install awscli

# または https://aws.amazon.com/cli/ からダウンロードする
```

**認証情報が設定されていない場合:**

```bash
aws configure
# 入力: AWS Access Key ID、Secret Access Key、デフォルトのリージョン、出力形式
```

**SSO を使っている場合:**

```bash
aws sso login --profile your-profile
```

**リージョンを確認します:** `aws configure` のリージョンは、Bedrock のモデルアクセスを有効にしたリージョン、およびデプロイ先のリージョンと一致している必要があります。

### チェック 5: Bedrock のモデルアクセス

```bash
aws bedrock list-foundation-models \
  --region $(aws configure get region) \
  --query 'modelSummaries[?contains(modelId, `claude`) && modelLifecycle.status==`ACTIVE`].modelId' \
  --output table
```

Claude のモデルが表示されない場合、またはアクセスエラーが出る場合:

1. AWS コンソール → Amazon Bedrock → Model access に移動する
2. "Manage model access" をクリックする
3. "Anthropic Claude" のモデルを有効にする
4. "Save changes" をクリックする — アクセスは通常、1 分以内に許可されます

**デフォルトのプロジェクトに必要なモデル:** デフォルトのモデルは、クロスリージョン推論プロファイルです（例: `global.anthropic.claude-sonnet-4-5-20250929-v1:0` — CLI はデフォルトで `global.` を生成します）。`global.` プレフィックスは、任意の商用リージョンにルーティングします。地域のプレフィックス（`us.`、`eu.`、`apac.`）は、推論をその地域内にとどめます。どのプレフィックスでも、そのプロファイルがカバーするすべての宛先リージョンで、モデルアクセスを有効にする必要があります。使用される正確なモデル ID は、`agentcore create` の後に `agentcore.json` で確認してください。

### チェック 6: IAM 権限

```bash
aws iam simulate-principal-policy \
  --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
  --action-names iam:CreateRole bedrock:InvokeModel \
  --resource-arns "*" \
  --query 'EvaluationResults[*].{Action:EvalActionName,Decision:EvalDecision}'
```

デプロイを動作させるには、次のものが必要です。

- `iam:CreateRole` — 実行ロールを作成するため
- `bedrock:InvokeModel` — Bedrock のモデルを呼び出すため
- `ecr:CreateRepository`、`ecr:PutImage` — コンテナビルド用
- `codebuild:StartBuild` — リモートビルド用

権限が不足している場合は、AWS の管理者に、`BedrockAgentCoreFullAccess` と `AmazonBedrockFullAccess` のマネージドポリシーを、IAM ユーザーまたはロールにアタッチするよう依頼してください。

### チェック 7: Docker（任意 — Container ビルドでのみ必要）

```bash
docker --version
docker info 2>&1 | head -5
```

Docker が必要なのは、`--build Container` を使う場合だけです。CodeZip ビルド（デフォルト）では、ローカルに Docker は不要で、AWS CodeBuild を使います。

**Docker が動作していない場合:**

- macOS: Docker Desktop を起動する
- Linux: `sudo systemctl start docker`

**Docker の代替:** AgentCore は、Podman と Finch もサポートしています。

---

## 結果の出力形式

結果は、明確なチェックリストとして報告します。

```
AgentCore Environment Check

✅ AgentCore CLI: 0.9.1
✅ Node.js: v20.11.0
✅ uv: 0.4.18
✅ AWS credentials: configured (account: 123456789012, region: us-east-1)
✅ Bedrock model access: Claude models enabled
⚠️  IAM permissions: missing iam:CreateRole — deploy will fail
❌ Docker: not running — needed for Container builds (optional)

Issues to fix:
1. IAM: Ask your admin to attach BedrockAgentCoreFullAccess to your user
2. Docker: Start Docker Desktop (only needed for Container builds)

All clear? Run `agents-get-started` to create your first project.
```

## 出力

- すべての前提条件の、合格/不合格のステータスを示したチェックリスト
- 失敗した各チェックに対する、正確な修正コマンド
- 作業を妨げているものと、任意のものの明確な区別
- 環境が正常な場合は、`agents-get-started` スキルへの案内
