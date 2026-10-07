# code-interpreter

AgentCore Code Interpreter ツールを追加して、エージェントがサンドボックス環境でコード（Python、JavaScript、TypeScript）を実行できるようにします。

## 使うタイミング

- エージェントが数値計算、データ分析、データ変換を行う必要があり、LLM の回答よりも計算結果のほうが信頼できる場合
- エージェントが回答としてコードを生成し、返す前にそのコードを実行して出力を検証したい場合
- エージェントが、S3 に永続化したいファイル（CSV、JSON、グラフ）を読み書きする必要がある場合
- 分離された、セッション単位のコードサンドボックスが必要な場合

この資料を使わないケース:

- Web ページを操作する場合 — [`browser.md`](browser.md) を参照してください
- 長時間稼働する任意のサービスを実行する場合 — Code Interpreter は短時間のコード実行用であり、サーバーのホスティング用ではありません
- *稼働中のエージェントセッション自身の microVM 内*でシェルコマンドを実行する場合 — これは `InvokeAgentRuntimeCommand` で、[`integrate.md`](integrate.md) で扱っています

## 考え方

Code Interpreter は、セッションごとに 1 つ、分離された microVM 内で動作する**マネージドサンドボックス**です。コードでは次のことができます。

- Python、JavaScript、TypeScript の実行
- ローカルファイルシステムでのファイルの読み書き（インラインアップロードで最大 100 MB、S3 経由で最大 5 GB）
- ネットワーク呼び出し（リソースでインターネットアクセスが有効な場合）
- プリインストール済みライブラリの利用（pandas、numpy、scikit-learn、torch など — 最新の一覧はドキュメントを参照）

セッションは、**セッション内ではステートフル**（同じセッション内の `execute_code` 呼び出し間で、変数とファイルが保持されます）、**セッション間では一時的**（新しいセッションを開始すると、ファイルシステムはクリーンな状態です）です。

## 前提条件

- エージェント環境の Python 3.10 以上
- `bedrock-agentcore` SDK
- リソース ARN に絞り込んだ、`bedrock-agentcore:*CodeInterpreter*` アクションに対する IAM 権限
- エージェントフレームワーク経由で呼び出す場合は、モデルへのアクセス権（フレームワークがモデルを呼び出して、コードを実行するかどうかを判断します）

IAM ポリシーの雛形:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CodeInterpreterAccess",
    "Effect": "Allow",
    "Action": [
      "bedrock-agentcore:CreateCodeInterpreter",
      "bedrock-agentcore:GetCodeInterpreter",
      "bedrock-agentcore:ListCodeInterpreters",
      "bedrock-agentcore:StartCodeInterpreterSession",
      "bedrock-agentcore:StopCodeInterpreterSession",
      "bedrock-agentcore:InvokeCodeInterpreter",
      "bedrock-agentcore:GetCodeInterpreterSession",
      "bedrock-agentcore:ListCodeInterpreterSessions"
    ],
    "Resource": "arn:aws:bedrock-agentcore:<REGION>:<ACCOUNT_ID>:code-interpreter/*"
  }]
}
```

アクション名は変更される可能性があるため、最新のものをドキュメントで確認してください。

## パス A — Code Interpreter を使う Strands エージェント（ほとんどの場合に推奨）

```python
from strands import Agent
from strands_tools.code_interpreter import AgentCoreCodeInterpreter

tool = AgentCoreCodeInterpreter(region="<REGION>")

agent = Agent(
    tools=[tool.code_interpreter],
    system_prompt=(
        "You are an assistant that validates claims with code. "
        "When asked to compute, calculate, or analyze, write Python and run it."
    ),
)

result = agent("What are the first 10 Fibonacci numbers?")
print(result.message["content"][0]["text"])
```

インストール: `pip install bedrock-agentcore strands-agents strands-agents-tools`

エージェントがコードを実行するタイミングを判断し、必要に応じてセッションを開始し、停止します。内部ではこのツールが AWS マネージドの `aws.codeinterpreter.v1` リソースを使用するため、リソースを作成する必要はありません。

**AgentCore Runtime のエントリーポイントに組み込む場合:**

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from strands import Agent
from strands_tools.code_interpreter import AgentCoreCodeInterpreter
from model.load import load_model
import os

app = BedrockAgentCoreApp()
REGION = os.getenv("AWS_REGION", "us-east-1")

@app.entrypoint
def invoke(payload, context):
    tool = AgentCoreCodeInterpreter(region=REGION)
    agent = Agent(
        model=load_model(),
        tools=[tool.code_interpreter],
        system_prompt="Validate computations with code.",
    )
    return {"response": str(agent(payload.get("prompt", "")))}

if __name__ == "__main__":
    app.run()
```

## パス B — プログラムによる実行のための SDK 直接利用

エージェントではなく、自分のコードが何を実行するかを決める場合に使います。ETL、データ変換、エージェント内部での検証に適しています。

```python
from bedrock_agentcore.tools.code_interpreter_client import code_interpreter_session

REGION = "us-east-1"

with code_interpreter_session(REGION) as session:
    # ステートフル: 変数はセッション内の呼び出し間で保持されます
    session.execute_code("import pandas as pd")
    session.execute_code("df = pd.DataFrame({'x': [1, 2, 3]})")
    result = session.execute_code("df.describe().to_string()")
    print(result.stdout)
```

コンテキストマネージャーが開始と停止を処理します。セッションをリークさせないでください。

**言語の選択** — デフォルトは Python です。JavaScript や TypeScript を使う場合は、`execute_code` に `language="javascript"` または `language="typescript"` を渡します（またはセッション開始時にランタイムを設定します）。サポートされているランタイムの最新情報は、ランタイム選択に関するドキュメントを参照してください。

## パス C — S3 アクセス付きのカスタム Code Interpreter

マネージドの `aws.codeinterpreter.v1` リソースには、S3 への書き込み権限がありません。成果物（グラフ、レポート、処理済みデータセット）を生成し、永続化したいエージェントの場合は、S3 アクセス権を持つ実行ロールを設定した**カスタム Code Interpreter** を作成します。

これは CreateCodeInterpreter の呼び出し（SDK/API。執筆時点では `agentcore` CLI からは利用できません）です。実行ロールの信頼ポリシーで `bedrock-agentcore.amazonaws.com` にロールの引き受けを許可し、権限ポリシーで成果物用バケットに対する `s3:PutObject` などのアクションを許可します。現在の `CreateCodeInterpreter` の形式と、信頼ポリシーの正確な書式は、ドキュメントで確認してください。

**同一アカウントの S3 に関するルール。** S3 バケットは、Code Interpreter リソースと**同じ AWS アカウント**にある必要があります。バケットポリシーを正しく設定していても、クロスアカウントのバケットは出力先としてサポートされません。`CreateCodeInterpreter` は検証エラーで失敗します。成果物が別のアカウントで必要な場合は、同一アカウントのバケットから後でレプリケートしてください。

## オブザーバビリティ

- **CloudWatch Logs** — 実行したコードの stdout/stderr と、セッションのライフサイクルイベント
- **CloudTrail** — `StartCodeInterpreterSession`、`InvokeCodeInterpreter`、`StopCodeInterpreterSession` のすべての呼び出し
- **メトリクス** — `AWS/BedrockAgentCore` 名前空間

## プリインストール済みライブラリ

マネージドの Python ランタイムには、`pandas`、`numpy`、`scipy`、`matplotlib`、`plotly`、`scikit-learn`、`torch`、`torchvision`、`statsmodels` のほか、データ分析・機械学習向けの数十個のライブラリが含まれています。ユーザーに「ライブラリ X はプリインストール済みです」と伝える前に、ドキュメントで最新の一覧を確認してください。一覧はプラットフォームの更新に伴って変わります。

プリインストールされていないライブラリは、セッション内で `install_packages(["your-lib==1.2"])` を呼び出します（または `execute_command` で `!pip install ...` を実行します）。インストールしたパッケージは、そのセッションの間だけ有効です。

## セッションのライフサイクル — 必ず閉じる

```python
# 正しい — コンテキストマネージャー
with code_interpreter_session(region) as session:
    session.execute_code("...")

# 正しい — 明示的なクライアントによる try/finally
client = CodeInterpreterClient(region=region)
client.start()
try:
    client.execute_code("...")
finally:
    client.stop()

# 誤り — リークしたセッションはタイムアウトまで残ります
```

デフォルトのセッションタイムアウトは 900 秒（15 分）、最大は 28800 秒（8 時間）です。リークしたセッションには費用がかかります。

## VPC モード

Code Interpreter は VPC をサポートしています。パターンは Runtime や Browser と同じです（サービスリンクロール、自分のサブネット、自分のセキュリティグループ）。[`vpc.md`](vpc.md) を参照してください。

**サンドボックスから公開インターネットへ**アクセスするには、Runtime と同様に、プライベートサブネットに NAT ゲートウェイが必要です。パブリックサブネットだけでは、Code Interpreter の ENI はインターネットにアクセスできません。コードが `pip install` で PyPI に接続する必要がある場合は、NAT を計画してください。

## よくある失敗

**StartCodeInterpreterSession での "Access denied":** リソース ARN に対するアクションが IAM に不足しています。`aws sts get-caller-identity` で、ポリシーをアタッチした ID と一致しているか確認してください。

**"ValidationException: Role does not have access to required S3 buckets":** S3 バケットが別のアカウントにあります。バケットを移動するか、同一アカウント内のステージング用バケットからレプリケートしてください。

**コードがタイムアウトする:** デフォルトの実行タイムアウトは短めです。長いジョブは分割するか、タイムアウトを延長したカスタム Code Interpreter を使用してください。30 分かかるトレーニングジョブを Code Interpreter で実行しようとしないでください。それは SageMaker や Batch のジョブの役割です。

**プリインストール済みと記載されているのに "Module not found" になる:** プリインストールの一覧は、`python` ランタイムと `nodejs` ランタイムで異なる場合があります。ランタイムの選択と、一覧が一致しているか確認してください。

## 出力

- どのパスが適しているか（Strands ツール、SDK 直接利用、S3 付きのカスタム）
- セッションのライフサイクルを処理した、動作するコード
- code-interpreter リソースに絞り込んだ IAM ポリシー

## 品質基準

- セッションは必ずコンテキストマネージャーか try/finally で囲み、リークさせない
- IAM はアカウント内の `code-interpreter/*` に絞り、`Resource: "*"` にしない
- S3 の出力先バケットは、Code Interpreter リソースと同じアカウントにある
- コードが Python でない場合は、言語・ランタイムの選択を明示する
