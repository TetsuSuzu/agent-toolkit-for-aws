# evals

AgentCore のエージェントに評価をセットアップします。1 回きりの品質チェックから、本番の完全なモニタリングのパイプラインまで扱います。

## 使うタイミング

- エージェントが良い回答をしているか知りたい
- ライブのトラフィックを継続的にモニタリングしたい
- 品質が下がったらビルドを失敗させる、CI/CD の品質ゲートが欲しい
- すでに実行した評価のスコアを解釈したい
- エージェントのバージョンを比較したい

特定の誤った回答のデバッグには使いません。そのためには `agents-debug` スキルを使ってください。

## 入力

`$ARGUMENTS` は省略可能です。指定された場合は、スキルの範囲を絞り込みます。

```
/evals                          # 対話形式 — 何をセットアップしたいかを質問します
/evals quick                    # 直近のセッションで、簡単な評価を実行します
/evals monitor                  # 継続的なオンラインモニタリングをセットアップします
/evals ci                       # CI/CD の品質ゲートのスクリプトを生成します
```

## 手順

### ステップ 1: 目的を理解する

質問します(または `$ARGUMENTS` から推測します)。

> 「何をしようとしていますか?
>
> 1. 直近のセッションで 1 回きりの評価を実行して、エージェントの状態を確認する
> 2. ライブのトラフィックの継続的なモニタリングをセットアップする
> 3. CI/CD パイプラインに品質ゲートを追加する
> 4. 特定のユースケース向けに、カスタムの評価器を作成する
> 5. すでに実行した評価のスコアを理解する」

### ステップ 2: 前提条件を確認する — そして、実際にデプロイが必要なものを把握する

`agentcore/agentcore.json` が存在する場合は、それを読みます。次の点を確認します。

- デプロイ済みのランタイムはあるか(常に必須というわけではありません。下記を参照)
- 設定済みの評価器はすでにあるか

**プロジェクトのコンテキストがない場合:** どのランタイムを評価したいかを尋ねます。スタンドアロンモードでは `--runtime-arn` を使えます。

**デプロイ済みのランタイムが実際に必要なものと、そうでないもの:**

| アクション | デプロイは必要か |
|---|---|
| 評価器の定義(`agentcore add evaluator`)— LLM-as-judge またはカスタムコード | **不要。** `agentcore.json` に書き込まれるだけです。 |
| LLM-as-judge の指示や評価尺度の作成と反復 | **不要。** テキストの編集です。保存したトレースや手作業の入力データで試せます。 |
| カスタムコードの評価器(`@custom_code_based_evaluator` 関数)のユニットテスト | **不要。** 関数をインポートして、`EvaluatorInput` のフィクスチャで呼び出します — 下の Path D を参照。 |
| CI/CD の品質ゲートのスクリプトの作成/ドライラン | スクリプト自体は**不要**。デプロイが必要なのは、スクリプト内の評価呼び出しを本番のトラフィックに対して行いたい場合のみ。 |
| **ローカル開発のトレースに対する `agentcore run eval`** | **不要。** `agentcore dev` は、デフォルトで OTEL のスパンを CloudWatch に出力します — 下の「ローカル開発の実行を評価する」を参照。 |
| **手作りのスパンによる `Evaluate` API**(boto3) | **不要。** `SessionSpans` を直接送信します。ランタイムはまったく不要です。 |
| 本番ランタイムのトレースに対する `agentcore run eval` | 必要 — デプロイ済みのランタイムが生成したトレースに対して動作する。 |
| `OnDemandEvaluationDatasetRunner`(SDK のデータセットランナー) | 必要 — ランナーはパイプライン内で AgentCore Runtime のエージェントを呼び出す。 |
| オンラインモニタリング(`agentcore add online-eval`) | 必要 — デプロイ済みのランタイムから継続的に取り込む。 |

**ローカル開発での評価のループは、現実的な選択肢です。** `agentcore dev` は、デプロイ済みのランタイムと同じように、OTEL を自動で計装して、スパンを CloudWatch に送ります。これはデプロイ済み専用の機能ではありません。自分のローカルでの呼び出しに対して評価器を反復でき、往復が短く、AWS CDK の手間もかかりません。

**データセットランナーとオンラインモニタリングでは、デプロイが本当に必要です。** これらの経路は、稼働中の AgentCore Runtime のエージェントを呼び出すか、そこから取り込むものです。ローカルでの同等の手段はありません。

**評価で前に進むために、先にフルデプロイするよう開発者に伝えないでください。** 定義、作成、ユニットテストはローカルでできます。以下の前提条件が満たされていれば、`agentcore run eval` の実行もローカルでできます。

#### ローカル開発の実行を評価する

要件:

1. **ローカルで AWS の認証情報が使えること**(例: スパンを届けたいアカウントに対して `aws sso login`)。
2. **アカウントで CloudWatch Transaction Search が有効になっていること。** 一度だけの設定です — CloudWatch コンソール(Settings → X-Ray traces → Transaction Search)か、次のコマンドで行います。

   ```bash
   aws xray update-trace-segment-destination --destination CloudWatchLogs
   ```

3. **OTEL がすでに有効であること。** `agentcore dev` は、デフォルトで AWS OpenTelemetry ディストリビューションを自動で計装します。`--no-traces` を渡している場合は、外してください。
4. 呼び出しの後に**約 10 秒待つこと** — CloudWatch の書き込みから読み取りまでのレイテンシーは、エンドツーエンドで約 10 秒です(トレースの読み取りと評価のクエリの両方をカバーします。取り込みのステップは 2 つではなく 1 つです)。

ループ:

```bash
# ターミナル 1 — OTEL を有効（デフォルト）にして、ローカル開発を開始する
agentcore dev

# ターミナル 2 — 何度か呼び出して、セッション ID を控える
agentcore dev --invoke "What's the weather in Seattle?" --stream
# または: dev が起動したら agentcore invoke "..."
# レスポンスまたはログから、セッション ID を控える。

# CloudWatch への取り込みのために約 10 秒待ってから、評価する
agentcore run eval \
  --runtime MyAgent \
  --session-id <session-id-from-local-run> \
  --evaluator "Builtin.Helpfulness"
```

評価器は AWS 上で動作しますが(マネージドの評価サービスなので、モデルの呼び出しはローカルではなくそこで行われます)、**評価対象のエージェントの実行は、手元のノート PC で行われました**。これは、評価器の指示や評価尺度を調整するための、最速の反復ループです。

#### 手作りのスパン(ランタイムはまったく不要)

最も短いユニットテストのループのために、または、エージェントを実行せずに、保存したスナップショットを評価したい場合は、自分で組み立てたスパンを使って、`Evaluate` API を直接呼び出します。

```python
import boto3

client = boto3.client("bedrock-agentcore", region_name="<REGION>")

response = client.evaluate(
    evaluatorId="Builtin.Helpfulness",
    sessionSpans=[
        # 最小限の形は、AgentCore のトレースの OTEL スパンのスキーマに一致する。
        # フィクスチャを作る最も簡単な方法: 実際のスパンを 1 つダウンロードして
        # （`agentcore traces get <traceId> --output trace.json`）、それを加工する。
        {"name": "agent.invoke", "attributes": {"gen_ai.prompt": "What's the weather?", "gen_ai.response.content": "Sunny, 72°F."}},
    ],
)
print(response["evaluatorResults"])
```

スパンのスキーマの全体とフィールドの一覧は、[`Understanding input spans`](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/understanding-input-spans.html) のドキュメントにあります。この方法は、1 回きりの評価には大げさですが、カスタムコードの評価器に、CloudWatch にまったく依存しない回帰テスト用のフィクスチャが必要な場合には、非常に価値があります。

---

### Path A: 1 回きりの評価を実行する

#### ステップ A1: 評価器を選ぶ

まず、組み込みの評価器から始めます。セットアップが不要で、最も一般的な品質の観点をカバーしています。

| 評価器 | レベル | 測定するもの |
|---|---|---|
| `Builtin.Helpfulness` | TRACE | 各応答がどれだけ役に立ったか |
| `Builtin.Correctness` | TRACE | 情報が事実として正確か(正解データをサポート) |
| `Builtin.Faithfulness` | TRACE | 応答が、与えられたコンテキストに基づいているか |
| `Builtin.ResponseRelevance` | TRACE | 応答が、尋ねられたことに答えているか |
| `Builtin.InstructionFollowing` | TRACE | エージェントが、システムプロンプトの指示に従っているか |
| `Builtin.Conciseness` | TRACE | 応答が適切に簡潔か |
| `Builtin.Coherence` | TRACE | 応答が論理的に一貫しているか |
| `Builtin.Refusal` | TRACE | エージェントが、範囲外のリクエストを適切に断ったか |
| `Builtin.ToolSelectionAccuracy` | TOOL_CALL | エージェントが、タスクに合った正しいツールを選んだか |
| `Builtin.GoalSuccessRate` | SESSION | エージェントが、ユーザーの目標を達成したか(正解データをサポート) |

**組み込みの評価器の名前は変わることがあります。** 現在の一覧は、AgentCore のドキュメントで確認してください。リリースごとに新しい評価器が追加されます。

**推奨:** 一般的な品質チェックには、`Builtin.Helpfulness` から始めます。タスクの完了には、`Builtin.GoalSuccessRate` を追加します。エージェントがツールを使う場合は、`Builtin.ToolSelectionAccuracy` を使います。比較対象となる正解データがある場合は、`Builtin.Correctness` または `Builtin.Faithfulness` を使います。

#### ステップ A2: 評価を実行する

```bash
# 直近のセッションに対して実行する（プロジェクトから自動で検出）
agentcore run eval --evaluator "Builtin.Helpfulness"

# 複数の評価器に対して実行する
agentcore run eval \
  --evaluator "Builtin.Helpfulness" \
  --evaluator "Builtin.GoalSuccessRate"

# 特定のランタイムに対して実行する（スタンドアロンモード、プロジェクト不要）
agentcore run eval \
  --runtime-arn arn:aws:bedrock-agentcore:us-east-1:123456789012:runtime/myagent-abc123 \
  --evaluator "Builtin.Helpfulness"

# 遡る期間を延長する（デフォルトは 7 日）
agentcore run eval --evaluator "Builtin.Helpfulness" --days 14
```

#### ステップ A3: 結果を解釈する

スコアは、0〜1 に正規化されます。

- **0.8〜1.0** — 良好。エージェントは、この観点で良好に動作しています。
- **0.6〜0.8** — 許容範囲。監視する価値はありますが、緊急ではありません。
- **0.6 未満** — 調査が必要。最近のトレースでパターンを確認してください。

結果は、`agentcore/.cli/eval-runs/` に保存されます。履歴を表示します。

```bash
agentcore evals history
agentcore evals history --limit 10
```

---

### Path B: 継続的なモニタリングをセットアップする

#### ステップ B1: 評価器を作成する(必要な場合)

継続的なモニタリングには、通常、組み込みの評価器で十分です。特定のユースケース向けのカスタムの評価器が必要な場合は、先に Path D を参照してください。

#### ステップ B2: オンライン評価の設定を追加する

```bash
agentcore add online-eval \
  --name my_quality_monitor \
  --runtime MyAgent \
  --evaluator "Builtin.Helpfulness" \
  --evaluator "Builtin.GoalSuccessRate" \
  --sampling-rate 5
```

**重要な命名ルール:** 設定の名前には、アンダースコアのみを使います — ハイフンは使えません。`my-monitor` は検証エラーで失敗し、`my_monitor` は動作します。

**サンプリングレートのガイダンス:**

- `1〜5` — 本番に適している(リクエストの 1〜5% を評価)
- `10〜20` — ステージング環境や、トラフィックの少ないエージェントに適している
- `100` — すべてのリクエストを評価する(開発/テスト専用。レイテンシーとコストが増える)

#### ステップ B3: デプロイして有効にする

```bash
agentcore deploy -y
```

オンライン評価の設定は `CREATING` の状態で始まり、デプロイ後、数秒以内に `ACTIVE` になります。

#### ステップ B4: 結果を表示する

結果は、CloudWatch Logs にストリーミングされます。

```
/aws/bedrock-agentcore/evaluations/results/<config-id>
```

AWS コンソールでの表示: CloudWatch → GenAI Observability → Bedrock AgentCore → Evaluations タブ。

CLI から評価のログをストリーミングします。

```bash
agentcore logs evals --runtime MyAgent --since 1h
agentcore logs evals --follow
```

**再デプロイせずに、一時停止/再開する:**

```bash
agentcore pause online-eval my_quality_monitor
agentcore resume online-eval my_quality_monitor
```

---

### Path C: CI/CD の品質ゲート

評価を実行して、品質がしきい値を下回ったらビルドを失敗させるスクリプトを生成します。

```bash
#!/bin/bash
# quality-gate.sh — CI/CD で、デプロイの後に実行する

set -e

RUNTIME="MyAgent"
EVALUATOR="Builtin.Helpfulness"
THRESHOLD="0.7"

echo "Running quality gate eval..."
result=$(agentcore run eval \
  --runtime "$RUNTIME" \
  --evaluator "$EVALUATOR" \
  --days 1 \
  --json)

score=$(echo "$result" | jq -r '.run.results[0].aggregateScore // empty')

if [ -z "$score" ]; then
  echo "⚠️  No eval data found. Has the agent been invoked recently?"
  echo "   Invoke the agent at least once, wait ~10 seconds, then re-run."
  exit 1
fi

echo "Quality score: $score (threshold: $THRESHOLD)"

if awk -v s="$score" -v t="$THRESHOLD" 'BEGIN{exit !(s<t)}'; then
  echo "❌ Quality gate FAILED: score $score < $THRESHOLD"
  exit 1
fi

echo "✅ Quality gate PASSED"
```

**注意:** CloudWatch の書き込みから読み取りまでのレイテンシーは、**エンドツーエンドで約 10 秒**です。同じ取り込みのステップで、トレースの読み取りと評価のクエリの両方が利用可能になり、追加のインデックス作成の待ち時間はありません。CI/CD では、統合テストの一部としてエージェントを呼び出し、その後、品質ゲートを実行する前に、短い `sleep 10`(余裕を見て `sleep 15`)を入れます。以前のスキルやドキュメントにあった `sleep 300` のパターンは、現在必要な時間の 30 倍です。

スタンドアロンモード(CI にプロジェクトのコンテキストがない場合):

```bash
agentcore run eval \
  --runtime-arn arn:aws:bedrock-agentcore:us-east-1:123456789012:runtime/myagent-abc123 \
  --evaluator "Builtin.Helpfulness" \
  --days 1 \
  --json
```

---

### Path D: カスタムの評価器

組み込みの評価器が、ドメインの正確さ、トーン、形式への準拠、ユースケース固有の安全性といった、特定の品質基準をカバーしていない場合は、カスタムの評価器を使います。

#### ステップ D1: 評価器の種類を選ぶ

- **LLM-as-a-judge** — LLM が、指示に照らして各応答を採点します。最も柔軟です。
- **コードベース** — Lambda 関数が、プログラムで応答を採点します。決定的なチェック(形式の検証、必須フィールドなど)に使います。

#### ステップ D2: LLM-as-a-judge の評価器を作成する

まず、適切なレベルを選びます。

- `SESSION` — 会話全体を評価する(目標の達成、全体的な品質)
- `TRACE` — 個々の応答を評価する(有用性、正確さ、トーン)
- `TOOL_CALL` — ツールの選択とパラメーターを評価する

評価器のレベルが追加されていないか、AgentCore のドキュメントで確認してください。リリースごとに新しいレベルが追加されることがあります。

```bash
agentcore add evaluator \
  --name ResponseQuality \
  --level TRACE \
  --model "global.anthropic.claude-sonnet-4-5-20250929-v1:0" \
  --instructions "Evaluate the assistant's response for helpfulness and accuracy. Context: {context}. Response to evaluate: {assistant_turn}" \
  --rating-scale 1-5-quality
```

注意: 上の評価器のモデル ID は例です。現在サポートされている評価器のモデル ID と、クロスリージョン推論プロファイルは、AgentCore のドキュメントで確認してください。

**レベルごとのプレースホルダーのルール:**

| レベル | 必須のプレースホルダー | 任意 |
|---|---|---|
| `SESSION` | `{context}` | `{available_tools}` |
| `TRACE` | `{context}` | `{assistant_turn}`、`{available_tools}` |
| `TOOL_CALL` | `{context}` | `{tool_turn}`、`{available_tools}` |

**評価尺度のプリセット**(`--rating-scale` に、文字列そのままで渡します):

- `1-5-quality` — Poor/Fair/Good/Very Good/Excellent(デフォルト)
- `1-3-simple` — Low/Medium/High
- `pass-fail` — Pass/Fail
- `good-neutral-bad` — Good/Neutral/Bad

**カスタムの評価尺度:**

```bash
--rating-scale "0:Incorrect:Factually wrong or misleading, 0.5:Partial:Partially correct, 1:Correct:Accurate and complete"
```

#### ステップ D3: コードベースの評価器を作成する(決定的なチェック用)

```bash
agentcore add evaluator \
  --name FormatChecker \
  --level TRACE \
  --type code-based \
  --lambda-arn arn:aws:lambda:<REGION>:<YOUR_ACCOUNT_ID>:function:check-response-format \
  --timeout 30
```

Lambda はトレースのコンテキストを受け取り、0〜1 のスコアを返す必要があります。SDK の `@custom_code_based_evaluator()` デコレーターを使うと、Lambda のイベントの解析とレスポンスの契約を代わりに処理してくれます。

```python
# lambda_function.py
from bedrock_agentcore.evaluation.custom_code_based_evaluators import (
    custom_code_based_evaluator,
    EvaluatorInput,
    EvaluatorOutput,
)

@custom_code_based_evaluator()
def handler(evaluator_input: EvaluatorInput, context) -> EvaluatorOutput:
    # evaluator_input.session_spans に、トレースのデータが入っている
    # 決定的なチェック(正規表現、スキーマの検証、ルールエンジンなど)を実装する
    response_text = _extract_response(evaluator_input.session_spans)

    if _matches_required_format(response_text):
        return EvaluatorOutput(value=1.0, label="Pass")
    return EvaluatorOutput(value=0.0, label="Fail", reasoning="Response did not match expected format")
```

デコレーターが、生の Lambda イベントの解析、トレース/スパンの ID の抽出、レスポンスのシリアライズを処理します。チェックは型付きの `EvaluatorInput` に対して書き、型付きの `EvaluatorOutput` を返します。

#### ステップ D3.5: デプロイする前に、評価器をローカルでユニットテストする

`@custom_code_based_evaluator` の関数は、単なる Python の関数です。直接インポートして、フィクスチャでロジックを検証します。デプロイも AWS の認証情報も不要です。

```python
# test_evaluator.py
from bedrock_agentcore.evaluation.custom_code_based_evaluators import EvaluatorInput
from lambda_function import handler  # 上でデコレートした関数

def _fake_input(response_text: str) -> EvaluatorInput:
    # ハンドラーが読む、最小限の EvaluatorInput の形を組み立てる。
    # より忠実なフィクスチャには、保存した実際のトレースを使う — 一度デプロイして呼び出した後に、
    # `agentcore traces get <traceId> --output trace.json` でダウンロードできる。
    return EvaluatorInput(
        session_spans=[{"attributes": {"gen_ai.response.content": response_text}}],
        # ...ご利用のレベルで SDK が期待する残りのフィールドを埋める
    )

def test_matches_format():
    out = handler(_fake_input('{"status": "ok"}'), context=None)
    assert out.value == 1.0

def test_rejects_free_text():
    out = handler(_fake_input("Here's your answer: ok"), context=None)
    assert out.value == 0.0
    assert "did not match" in (out.reasoning or "").lower()
```

`pytest test_evaluator.py` で実行します。フィクスチャが通るまでロジックを反復してください。デプロイするのは**その後**です。デプロイのステップは、Lambda を AgentCore に接続するためのもので、チェックのデバッグのためのものではありません。

**LLM-as-judge** の評価器には、同等のユニットテストの手段はありません(モデルの呼び出しは評価サービスで行われます)。ただし、`agentcore add evaluator` の前に、Bedrock コンソールや使い捨てのスクリプトでプロンプトをドライランすることで、保存したトレースに対して指示を反復できます。

#### ステップ D4: デプロイして、実際のトレースに対して実行する

```bash
agentcore deploy -y
agentcore run eval --evaluator ResponseQuality --days 7
```

**評価器の名前のルール:** 英数字とアンダースコアのみ、最大 48 文字。ハイフンは使えません。

---

## トラブルシューティング

### "No spans found for session"

- エージェントを呼び出してから約 10 秒待つ — CloudWatch の書き込みから読み取りまでは、エンドツーエンドで約 10 秒です(それ以外に、評価用の別個のインデックス作成のステップはありません)
- エージェントのデプロイ時に、オブザーバビリティが有効だったことを確認する
- 遡る期間を延長する: `--days 14` または `--days 30`

### "No agent specified" またはエージェント ID が見つからない

- AgentCore のプロジェクトのディレクトリ内から実行する、または
- `--runtime-arn` を使って、エージェントを明示的に指定する

### オンライン評価の設定が CREATING のまま止まる

- `agentcore status --type online-eval` を実行して、状態を確認する
- 通常は、デプロイから 30 秒以内に解決する

### `remove evaluator` が失敗する

- オンライン評価の設定が、この評価器を参照している
- 先に、オンライン評価の設定を削除する: `agentcore remove online-eval --name <name>`
- その後、評価器を削除する

## クロスリージョン推論(データレジデンシー)

組み込みの評価器と LLM-as-judge の評価器は、どちらもデフォルトで**クロスリージョン推論**を使います。評価対象のデータはプライマリリージョンにとどまりますが、判定モデルを実行する推論の呼び出しは、同じ地域内の別の AWS リージョンで実行される場合があります(例: `us-east-1` → `us-east-2`/`us-west-2`。EU は EU 内にとどまります)。

追加の費用はかからず、ログに推論を行ったリージョンは含まれません。ただし、データレジデンシーの規則で推論を単一のリージョンに固定する必要がある場合は:

- **組み込みの評価器:** AgentCore が管理しており、設定されたとおりにクロスリージョン推論を使います。単一リージョンでの推論が必要な場合は、代わりにカスタムの評価器を使います。
- **カスタムの LLM-as-judge の評価器:** `--model` に、クロスリージョン推論プロファイルの ID ではなく、リージョン固有のモデル ID を指定して、モデルを固定します。お使いのリージョンで現在使える、単一リージョンのモデル ID は、ドキュメントで確認してください。
- **コードベースの評価器:** 影響を受けません。Lambda は、デプロイした場所で実行されます。

現在の地域とリージョンの対応は、ここに固定して書くのではなく、[クロスリージョン推論](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/cross-region-inference.html)を参照してください。リリースごとに拡大していきます。

## データセットランナーと `agentcore run eval` の使い分け

2 つの異なるワークフローのための、2 つの異なるツールです。開発者は、よくこの 2 つを混同します。

| やりたいこと | ツール | 実行される場所 |
|---|---|---|
| 最近の実行から、1 つのセッションまたはトレースを評価する | `agentcore run eval --session-id <id>` | CLI、CloudWatch に取り込まれたスパンに対して |
| 過去 N 日間の*すべて*を評価し、スコアのドリフトを追跡する | `agentcore run eval --days 7` | CLI、CloudWatch に対して |
| 精選したベンチマーク/回帰スイート(20〜500 のシナリオ、CI/CD)を実行する | `OnDemandEvaluationDatasetRunner`(SDK) | 自分の Python プロセス。呼び出し + 待機 + 評価を取りまとめる |
| 本番のすべての呼び出しが、品質のしきい値を満たしているか確認する | `agentcore add online-eval` | プラットフォーム、継続的なサンプリング |

**`agentcore run eval` を使う**のは、評価器を反復している場合、特定の回帰を調査している場合、最近のトラフィックに対して品質ゲートを実行する場合です。高速で安価で、エージェント自体は呼び出しません。既存のトレースを採点するだけです。

**`OnDemandEvaluationDatasetRunner` を使う**のは、期待する応答/軌跡/アサーションを持つシナリオのデータセットがあり、それをバッチとして実行したい場合です。ランナーは、シナリオごとに**エージェントを呼び出し**、テレメトリの取り込みを待ち(デフォルトは 180 秒。シナリオごとではなく、実行ごとに 1 回だけ)、その後評価します。これにはデプロイ済みのランタイムが必要です。典型的な用途は、新しいバージョンを昇格する前の、CI での回帰テストのパックです。

**オンライン評価を使う**のは、サンプリングレートでの継続的な本番のモニタリングです。ベンチマークとは別のものです。

## 出力

- 評価の実行、またはモニタリングのセットアップのための CLI コマンド
- 品質ゲートのスクリプト(CI/CD のパス)
- 評価器の設定(カスタムの評価器のパス)
- 既存の結果を確認する場合は、スコアの解釈
