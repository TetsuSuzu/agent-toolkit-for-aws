# migrate

既存の Amazon Bedrock Agent を AgentCore Runtime に移行します。

## 使うタイミング

- 既存の Bedrock Agent（Bedrock コンソールまたは API で作成したもの）があり、AgentCore Runtime で実行したい
- 既存のエージェントに AgentCore の機能（Memory、Gateway、オブザーバビリティ）を追加したい
- 宣言型の Bedrock Agents モデルから、コードファーストのフレームワークに移行したい

## 入力

`$ARGUMENTS` は省略可能です。

```
/migrate                    # 対話形式 — 移行を順に案内します
/migrate strands            # Strands フレームワークを対象に移行します
/migrate langgraph          # LangGraph フレームワークを対象に移行します
```

## 移行で行われること

`agentcore create --type import` コマンドは、既存の Bedrock Agent の設定を読み取り、その動作をコードファーストのフレームワークで再現する AgentCore プロジェクトを生成します。具体的には次のとおりです。

- **システムプロンプト** → 生成される `main.py` にコピーされます
- **アクショングループ（Lambda ベース）** → `--type lambda-function-arn` を指定した Gateway ターゲットに変換されます
- **ナレッジベース** → システムプロンプトで参照されますが、検索の接続は手動で行う旨の注記が付きます（AgentCore は KB のバインディングを自動でインポートしません）
- **ガードレール** → コメントに記載されますが、自動では変換されません（AgentCore は Bedrock Guardrails ではなく Cedar ポリシーを使用します）
- **エージェントのエイリアス/バージョン** → インポートの対象は、ドラフトではなく特定のエイリアスです

移行で行われ**ない**こと:

- 元の Bedrock Agent の削除や変更は行いません — 移行元のエージェントは動き続けます
- 会話履歴やセッション状態は移行されません
- Bedrock Guardrails を Cedar ポリシーに変換しません（認可モデルが異なります）
- ナレッジベースの検索は自動で接続されません — ツールまたは SDK の直接呼び出しとして追加する必要があります

## 前提条件

1. Bedrock Agent が存在し、少なくとも 1 つのエイリアスを持っていること
2. AWS 認証情報に `bedrock:GetAgent`、`bedrock:GetAgentAlias`、`bedrock:ListAgentActionGroups` の権限があること
3. エージェント ID、エイリアス ID、リージョンが分かっていること

## 手順

### ステップ 1: インポートを実行する

```bash
agentcore create \
  --type import \
  --agent-id <AGENT_ID> \
  --agent-alias-id <ALIAS_ID> \
  --region <REGION> \
  --name <ProjectName> \
  --framework Strands
```

`--framework` フラグで、生成されるプロジェクトが使用するコードファーストのフレームワークが決まります。Bedrock Agent の動作に最も近い対応になるため、Strands を推奨します。

**プロジェクト名のルールが適用されます:** 最大 23 文字、英数字のみ、先頭は英字。

### ステップ 2: 生成されたプロジェクトを確認する

```bash
cd <ProjectName>
cat app/<AgentName>/main.py
cat agentcore/agentcore.json
```

確認する項目:

- システムプロンプトが、元のエージェントの指示と一致している
- アクショングループが、`agentcore.json` の Gateway ターゲット（`agentCoreGateways` の下）として現れている
- モデル ID が、対象のリージョンで正しい

### ステップ 3: 移行で対応されない部分を補う

**ナレッジベース:** Bedrock Agent がナレッジベースを使っていた場合、2 つの選択肢があります。

1. **boto3 経由で KB を使い続ける** — エージェントのコードから、ツールとして `bedrock-agent-runtime:RetrieveAndGenerate` または `Retrieve` を直接呼び出します
2. **AgentCore Memory に置き換える** — KB がユーザー固有のコンテキストに使われていた場合は、SEMANTIC 戦略の AgentCore Memory のほうが適している可能性があります。[memory.md](memory.md) を参照してください。

**ガードレール → Cedar ポリシー:** Bedrock Guardrails（コンテンツフィルター、拒否トピック、単語フィルター）は、Cedar ポリシーと 1 対 1 には対応しません。Cedar ポリシーは、*エージェントがどのツールを、どのパラメーターで呼び出せるか*を制御する認可ルールであり、コンテンツフィルターではありません。コンテンツのフィルタリングが必要な場合は、ガードレールのロジックをエージェントのコード（前処理/後処理）に残すか、Bedrock Guardrails を単独の API 呼び出しとして使用してください。

**カスタムオーケストレーション:** Bedrock Agent がカスタムオーケストレーション（return-of-control、カスタム Lambda オーケストレーター）を使っていた場合は、そのロジックをフレームワークのネイティブなパターン（Strands のツールチェーン、LangGraph のグラフノードなど）で再構築する必要があります。

### ステップ 4: ローカルでテストしてデプロイする

```bash
# ローカルでテストする（Memory と Gateway はまだ利用できません）
agentcore dev

# 準備ができたらデプロイする
agentcore deploy -y

# 確認する
agentcore invoke "Hello, what can you do?"
agentcore status
```

### ステップ 5: トラフィックを切り替える

AgentCore のエージェントが正しく動作したら:

1. アプリケーションを更新して、Bedrock Agent ではなく AgentCore Runtime を呼び出すようにします
2. 呼び出しパターン（SigV4、JWT、SDK）は [integrate.md](integrate.md) を参照してください
3. 確信が持てるまで、元の Bedrock Agent をフォールバックとして稼働させておきます
4. AgentCore のエージェントが本番で安定稼働してから、Bedrock Agent を削除します

## 移行でよくある問題

**"Model not available in target region"**
インポートされたエージェントが、AgentCore のデプロイ先リージョンで利用できないモデル ID を参照している可能性があります。`model/load.py` を編集して、クロスリージョン推論プロファイルか、そのリージョンで利用できるモデルを使用してください。

**"Action group Lambda in a different region"**
Gateway ターゲットは別リージョンの Lambda 関数を呼び出せますが、レイテンシーが増加します。Lambda を AgentCore のエージェントと同じリージョンにデプロイするか、レイテンシーとのトレードオフを受け入れることを検討してください。

**"Agent behavior differs after migration"**
最も多い原因は、Bedrock Agent のオーケストレーションと、コードファーストのフレームワークとの間のプロンプト形式の違いです。Bedrock Agent はツールの結果の周囲に構造化された XML を挿入しますが、Strands や LangGraph は異なる形式を使います。これを補うように、システムプロンプトを調整してください。

## 出力

- Bedrock Agent の動作を再現する、動作する AgentCore プロジェクト
- 自動で変換されたものと、手作業が必要なものの一覧
- 旧エージェントから新エージェントへトラフィックを切り替えるためのガイダンス
