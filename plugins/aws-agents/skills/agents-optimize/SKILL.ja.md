---
name: agents-optimize
description: >
  エージェントの品質やパフォーマンスを測定または改善するときに使います — 
  評価、オンラインモニタリング、CI/CD の品質ゲート、オブザーバビリティ、
  コスト最適化のセットアップ。トリガー: "evaluate my agent"、"add evaluator"、
  "measure quality"、"quality gate"、"run evals"、"agent too slow"、
  "why is it slow"、"reduce latency"、"set up observability"、"CloudWatch
  dashboard"、"how much does my agent cost"、"cost optimization"、"logs
  not showing up"、"logs missing"、"spans not found"、"eval failing"、
  "eval error"、"dev traces"、"local traces"、"agentcore dev traces"、
  "traces to CloudWatch"。
  エラーやクラッシュのデバッグには使いません — agents-debug を使います。遅いが
  正しく動作している場合はここで扱い、壊れている場合は debug に振り分けます。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# optimize

評価、モニタリング、オブザーバビリティを通じて、AgentCore のエージェントの品質を測定し、改善します。

## 使うタイミング

- エージェントが良い回答をしているか知りたい
- 本番で、継続的な品質モニタリングを設定したい
- CI/CD パイプラインに品質ゲートを追加したい
- ログ、メトリクス、トレースを通じて、エージェントの動作を理解したい
- CloudWatch のダッシュボードや X-Ray のトレーシングを設定したい

次の場合は使わないでください:

- 特定の壊れたエージェントのデバッグ(誤った回答、エラー)→ `agents-debug` を使います
- 本番のセキュリティ堅牢化(IAM、認証)→ `agents-harden` を使います

## 入力

`$ARGUMENTS` には次のものを指定できます。

- 評価の目標: "add a quality gate"、"set up monitoring"
- オブザーバビリティの目標: "set up CloudWatch dashboard"、"understand my traces"
- 特定の評価器(Evaluator): "llm-as-a-judge"、"code-based"
- 空 — スキルがプロジェクトのコンテキストに基づいて案内します

## 手順

### ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。

### ステップ 1: プロジェクトのコンテキストを読み取る

`agentcore/agentcore.json` を読み、既存の評価器、オンライン評価の設定、エージェントのセットアップを把握します。

`agentcore/agentcore.json` が見つからない場合:
> 「このスキルには AgentCore のプロジェクトが必要です。`agents-get-started` で作成してください。」

### ステップ 2: ワークフローを判断する

| 開発者の意図 | アクション |
|---|---|
| 品質の測定、評価器の追加、評価の実行、CI/CD のゲート、オンラインモニタリング | [`references-ja/evals.md`](references-ja/evals.md) を読み込み、そのワークフローに従う |
| オブザーバビリティ、CloudWatch、X-Ray、ログ、メトリクス、ダッシュボードのセットアップ | [`references-ja/observability.md`](references-ja/observability.md) を読み込み、そのワークフローに従う |
| AgentCore のコストを理解する、または削減する | [`references-ja/cost.md`](references-ja/cost.md) を読み込む |
| 両方 — 「エージェントを理解して改善したい」 | オブザーバビリティのセットアップから始め、その後で評価を追加する |

### ステップ 3: 読み込んだリファレンスに従う

リファレンスのファイルに、完全な手順が書かれています。順番に従ってください。

### 相互参照

- 評価をセットアップしたら、本番準備のために `agents-harden` を提案する
- 評価の結果でエージェントの問題が明らかになった場合は、根本原因の分析のために `agents-debug` を提案する
- 開発者が先に機能を追加する必要がある場合は、`agents-build` を提案する

## 出力

ワークフローによって異なります。具体的な出力は、読み込んだリファレンスを参照してください。

## 品質基準

- 評価器の設定が、有効な CLI フラグのみを使っている
- オンライン評価のサンプリングレートが適切である(議論なしに、本番で 100% にしない)
- CI/CD の品質ゲートに、明確な合格/不合格のしきい値がある
- オブザーバビリティのセットアップに、トレーシングとロギングの両方が含まれている
- 開発者が、評価データの遅延を理解している: **書き込みから読み取りまで、エンドツーエンドで約 10 秒** — 1 回の取り込みのステップが、トレースの読み取りと評価のクエリの両方をカバーします。別個のインデックス作成の待ち時間はありません
