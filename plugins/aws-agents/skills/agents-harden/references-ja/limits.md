# limits

AgentCore Runtime のクォータを理解し、どれに当たっているかを診断し、必要なときに引き上げを申請します。

## 使うタイミング

- エージェントがスロットリングされている、またはクォータに関連するエラーを返している
- `ServiceQuotaExceededException: maxVms limit exceeded` が発生している
- Service Quotas コンソールでクォータを検索したが、見つからなかった
- ローンチを計画しており、クォータが妨げにならないようにしたい
- クォータの引き上げを申請しようとしていて、一度で正しく申請したい

## ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。それより古い場合は、`agentcore update` を実行します。

---

## どの上限に当たっているのか

表示されているエラーから、該当するクォータを特定します。

### 呼び出しのレート

**エラーの形:**

```
ThrottlingException: Rate exceeded
HTTP 429 Too Many Requests
```

`InvokeAgentRuntime` または `InvokeAgentRuntimeWithWebSocketStream` で発生します。

**意味:** アカウント内の 1 つのエージェントエンドポイントに対する、1 秒あたりの呼び出しが多すぎます。デフォルトは、エージェントごと、アカウントごと、リージョンごとに 25 TPS です。調整可能です。

**引き上げを申請する前に:**

- 指数バックオフとジッターを使った、クライアント側のリトライを追加する — スロットリングのスパイクは、通常は一時的なものです
- トラフィックがバーストに集中しているか、分散しているかを確認する — 平均のレートがずっと下回っていても、同じミリ秒に 100 件のリクエストが来ると、TPS の上限に達します
- そのレートが、長期的に本当に必要なものであれば、引き上げを申請する(以下の手順)

---

### 同時 VM / アクティブセッションの上限

**エラーの形:**

```
ServiceQuotaExceededException: maxVms limit exceeded
```

**意味:** アカウントで、AgentCore Runtime のアクティブな microVM が同時に多すぎます。AgentCore のドキュメントでは、このクォータは**「アカウントあたりのアクティブセッションのワークロード」**と呼ばれています。デフォルトは、us-east-1 と us-west-2 で 1,000、その他のリージョンで 500 です。調整可能です。

**重要 — 引き上げを申請する前にお読みください:**

CloudWatch の「同時アクティブセッション」のメトリクスは、稼働中の VM の数と同じではありません。`maxVms` のクォータは、呼び出しを完了したもののまだ回収されていないセッションを含めて、アカウント内で稼働しているすべての microVM を数えます。CloudWatch の同時実行のメトリクスが 50 を示していても、実際に稼働中の VM の数は 500 ということがあります。

これに当たる顧客の大半の本当の根本原因は、真の同時実行ではなく、セッションのライフサイクルです。まず、次の点を確認してください。

1. **各呼び出しの完了後に、`StopRuntimeSession` を呼び出していますか?** 呼び出していない場合、VM は `idleRuntimeSessionTimeout`(デフォルトは 900 秒 / 15 分)が切れて回収されるまで残ります。控えめなリクエストレートでも、VM は積み上がります。

2. **関連するリクエスト間で、セッション ID を再利用していますか?** リクエストごとに一意のセッション ID を使うと、リクエストごとに新しい環境ができます。セッション ID を再利用すると、以降のリクエストが同じ環境にルーティングされ、VM の総数が低く保たれます。

3. **`idleRuntimeSessionTimeout` は、ワークロードに適していますか?** 短時間のリクエストでデフォルトの 900 秒のタイムアウトのままだと、各 VM は、最後のリクエストの後 15 分間、スロットを占有します。`agentcore/agentcore.json` のランタイムの `lifecycleConfiguration` を編集して、`agentcore deploy` を実行し、短くしてください。

3 つすべてを確認してもなお上限に当たる場合は、引き上げを申請してください。

パターンとコードのスニペットは、`agents-harden` の SKILL.md(Session lifecycle management)を参照してください。

---

### 新規セッションの作成レート

**エラーの形:**

```
HTTP 429 Too Many Requests
```

**意味:** 新しいセッションの作成のレート(エンドポイントごと、アカウントごと)です。デフォルトは、コンテナのデプロイで 100 TPM、コードの直接デプロイで 25 TPS です。調整可能です。

**引き上げを申請する前に:**

- 可能な限りセッション ID を再利用する — 新しいセッションが少なければ、このクォータへの負荷も減ります
- 可能であれば、トラフィックを分散する — 新規セッションのリクエストのバーストは、定常的なトラフィックよりも、レート制限に強く当たります

---

### Memory 操作のレート

**エラーの形:**

```
ThrottlingException
```

`CreateEvent`、`RetrieveMemoryRecords`、`ListEvents` などの Memory API で発生します。

**意味:** AgentCore Memory には、操作ごとのレート制限があります。エージェントのコードは通常、読むよりも多く書くため、`CreateEvent`(デフォルトは 10 TPS)に最も当たりやすいです。Memory API の上限のほとんどは調整可能です。

**引き上げを申請する前に:**

- 指数バックオフを使った、クライアント側のリトライを追加する
- 同じ内容を繰り返し書き込んでいないか確認する(たとえば、事実ごとに 1 回ではなく、ターンごとに書き込んでいないか)
- 長期 Memory の抽出については、`Bedrock-AgentCore` 名前空間の CloudWatch メトリクス `TokenCount` を監視する — デフォルトは、アカウントあたり 1 分間に 150,000 トークンです(調整可能)

---

### Gateway のターゲット数またはリクエストレート

**エラーの形:**

```
ValidationException: Too many targets for gateway
ThrottlingException
```

**意味:** 1 つの Gateway には、ターゲットの数(Gateway あたりのデフォルトは 100、調整可能)、ターゲットあたりのツール数(デフォルトは 1,000、調整可能)、呼び出しレートに上限があります。

**引き上げを申請する前に:**

- 可能な限りツールを統合する — ツールごとに 1 つの Lambda を用意するより、複数のツール定義を持つ 1 つの Lambda のほうが効率的です
- 論理的に分かれたツールのグループがある場合は、複数の Gateway に分割する

---

### Code Interpreter / Browser のセッションの上限

**エラーの形:**

```
ServiceQuotaExceededException
```

項目名が、Code Interpreter または Browser のセッションを参照しています。

**意味:** 組み込みツールの同時セッションの上限です。

**引き上げを申請する前に:**

- 作業の完了時に、セッションが明示的に終了されていることを確認する
- まだカウントされているかもしれない、以前の実行から残った孤立セッションがないか確認する

---

## 現在のクォータの値を調べる場所

1. **正式なリファレンス:** AgentCore の上限に関するドキュメントページ — https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/bedrock-agentcore-limits.html — は、すべてのクォータについて、デフォルト値と調整可能かどうかを一覧にしています。**Adjustable** の列が、正しい情報源です。`awsknowledge` MCP サーバーが利用できる場合は、`aws___search_documentation` ツールで現在のクォータの値を調べられます。古い可能性のあるリンクに頼らずに、最新のドキュメントを直接取得できます。

2. **Service Quotas コンソール:** https://console.aws.amazon.com/servicequotas/home → **AWS services** → "Amazon Bedrock AgentCore" を検索する。現在適用されているクォータの値を表示し、そこから直接、引き上げを申請できます。現在、調整可能な AgentCore のクォータは、ほぼすべてここで利用できます — これがデフォルトの経路です。

3. **`aws service-quotas list-services`** / **`list-service-quotas`:** プログラムから確認する方法 — `aws service-quotas list-services` を実行して、"agentcore" または "bedrock" を grep し、現在のサービスコードを見つけたら、`list-service-quotas --service-code <code>` でクォータを一覧表示します。

---

## クォータの引き上げを申請する方法

Service Quotas コンソールを使います。調整可能な AgentCore のクォータは、ほぼすべてそこにあります。AWS サポートへの直接のケースが必要になるのは、お使いのリージョンで特定のクォータがコンソールに表示されない、まれな場合だけです。

### Path 1 — Service Quotas コンソール(こちらを使う)

1. 引き上げが必要なリージョンで、https://console.aws.amazon.com/servicequotas/home を開く。クォータはリージョンごとです — 先に進む前に、右上で正しいリージョンを選択してください。
2. ナビゲーションペイン → **AWS services** → **Amazon Bedrock AgentCore** を検索して選択する。
3. 一覧からクォータを探す。**Adjustable** の列が、引き上げを申請できるかを示します。
4. クォータを選択 → **Request increase at account-level**(そのクォータでリソースレベルが利用できる場合は **resource-level**)。
5. 新しい値を入力する(現在適用されている値より大きい必要があります)→ **Request**。
6. 状態を **Request history** タブか、ナビゲーションペインの **Dashboard** で追跡する。状態が **Pending** から **Quota requested** に変わると、サポートのケース番号が割り当てられます — コンソールからそのケースを開いて、進捗を確認できます。

**その後に起きること:**

- 小さな引き上げは、数分から数時間以内に自動で承認されることが多いです。
- 大きな引き上げは、AWS サポートにエスカレーションされ、時間がかかります(規模とクォータによって、数時間から数日)。
- サポートは、承認、部分的な承認、却下のいずれかを行えます。却下された場合は、コンソールのメッセージで理由が説明されます。より詳しい根拠を添えて、新たに申請できます。

**CLI での同等の操作** — スクリプト化したワークフロー向け:

```bash
# AgentCore の Service Quotas のサービスコードを調べる（サービスの API プレフィックスに
# 従う — 正確なコードを確認するために、一度実行する）
aws service-quotas list-services \
  --region <REGION> \
  --query "Services[?contains(ServiceName, 'AgentCore') || contains(ServiceCode, 'agentcore')]"

# そのサービスコードのクォータを一覧表示する
aws service-quotas list-service-quotas \
  --service-code <SERVICE_CODE> \
  --region <REGION>

# 引き上げのリクエストを送信する
aws service-quotas request-service-quota-increase \
  --service-code <SERVICE_CODE> \
  --quota-code <QUOTA_CODE> \
  --desired-value <NEW_VALUE> \
  --region <REGION>

# 状態を確認する
aws service-quotas list-requested-service-quota-change-history-by-quota \
  --service-code <SERVICE_CODE> \
  --quota-code <QUOTA_CODE> \
  --region <REGION>
```

CLI でリクエストを送信するアカウントには、`ServiceQuotasFullAccess`(または同等のもの)と、Service Quotas があなたに代わってサポートのケースを作成できるようにするための `iam:CreateServiceLinkedRole` が必要です。

### Path 2 — AWS Support Center のケース(例外的なケース)

必要な特定のクォータが、お使いのリージョンの Service Quotas コンソールに表示されない場合、またはコンソールが "This quota can't be increased from this console." と返す場合にのみ必要です。これはまれなので、まずコンソールを確認してください。

1. https://console.aws.amazon.com/support を開く。AWS コンソールの **?** ヘルプアイコン → **Support Center** からもアクセスできます。
2. **Create case**。
3. **Case type** → **Service quotas**。
4. **Service** → **Service Limit increase**。
5. **Category** → AgentCore のサービス(例: "Amazon Bedrock AgentCore Runtime")を選択する。特定の AgentCore のカテゴリが一覧にない場合は、最も近いものを使い、正確なクォータ名を説明に記載する。
6. **Region** → 引き上げが必要な AWS リージョンを選択する。**Add another limit** を選ぶと、複数のリージョンへの同じ引き上げを 1 つのケースで申請できます。
7. **Description** — 審査担当者が必要とするすべてを含める(下のフィールドを参照)。
8. **Contact method**(Web、Chat、Phone)を選択 → **Submit**。

### どの経路でも必要な情報

Service Quotas コンソールの根拠のフィールドでも、サポートケースの説明でも、審査担当者が承認できるだけの情報を提供してください。

- **AWS アカウント ID**(引き上げが必要なアカウント)
- **リージョン** — 上限はリージョンごと。複数のリージョンをカバーするサポートケースの場合は、必要なすべてのリージョンを列挙する
- **クォータ名** — AgentCore の上限に関するドキュメントの正確な名前に合わせる(例: "Active session workloads per account"、"InvokeAgentRuntime API rate, per agent, per account")
- **現在の値 → 希望する値** — 具体的に(例: "25 → 100")
- **Agent Runtime ID** または ARN — このリクエストが対象とするもの
- **ユースケース** — エージェントが何をするか、トラフィックのパターンについて 1〜3 文(一部のクォータでは、持続的かバーストかが重要)
- **想定されるピーク** — 範囲ではなく、実際の数値(ピークの TPS、同時セッション数など)
- **ビジネスへの影響** — 現在の上限で何がブロックされるか(例: 「X 日の GA ローンチがブロックされる」)
- **期限 / 必要な日付**

### コピー&ペーストできる根拠のテンプレート

Service Quotas の根拠のフィールド、またはサポートケースの説明に、これを貼り付けます。

```
Account ID: <12-digit account>
Region(s): <comma-separated>
Quota name: <exact name from AgentCore limits docs>
Current value: <N>
Requested value: <N>
Agent Runtime ID(s): <comma-separated agentRuntimeId or ARN values>

Use case:
  <1–3 sentences describing what the agent does and the traffic pattern>

Expected peak:
  <specific number — peak TPS, concurrent sessions, etc.>

Business impact if not raised:
  <what happens to your workload at the current limit>

Need-by date: <date>
```

### 承認が早くなること

- 「できるだけ高く」ではなく、具体的な数値
- トラフィックのパターンの説明 — 持続的かバーストか
- ある場合は、ローンチ前の負荷テストの数値
- 本番のローンチ日の明記

### 承認が遅くなること

- 上記の軽減策(`StopRuntimeSession`、セッションの再利用、バッチ処理、リトライ)を試す前に、引き上げを申請する
- 「念のため」すべてのクォータを、大きなキリの良い数字にして申請する
- 正確なクォータ名がない — 審査担当者は、どのサービスのどのクォータなのかを知る必要がある
- 1〜2 つのリージョンしか必要ないのに、すべてのリージョンで引き上げを申請する

---

## 申請する前に: 簡単なトリアージ

まず、このリストを順に確認してください。「上限に当たっている」問題の大半は、実際の引き上げを必要とせずに、これらのステップのどこかで解決します。

- [ ] そのエラーは、本当にクォータのエラーか?(例外のクラスとコードを確認する — すべての `Exception` がスロットリングではない)
- [ ] 一時的なスロットリングに対して、指数バックオフを使ったクライアント側のリトライがあるか?
- [ ] `maxVms` の場合: 各呼び出しの後に、`StopRuntimeSession` が呼び出されているか?
- [ ] `maxVms` の場合: 関連するリクエスト間で、セッション ID が再利用されているか?
- [ ] `maxVms` の場合: `idleRuntimeSessionTimeout` が、ワークロードに適切に設定されているか?
- [ ] Memory の書き込みの場合: 可能な限りバッチ処理していて、重複した書き込みをしていないか?
- [ ] トラフィックはバーストしているか?呼び出し元で平準化できないか?
- [ ] 現在のクォータが本当に問題か、それとも下流の依存先が本当のボトルネックか?

上記をすべて確認してもなお引き上げが必要な場合は、Service Quotas コンソールから申請してください。

## 出力

- エラーに基づいて、当たっている特定のクォータを特定する
- 引き上げを申請する前に試す軽減策
- 申請の経路: Service Quotas コンソール(まれに、サポートのケース)— 貼り付けられる、記入済みの根拠付き
