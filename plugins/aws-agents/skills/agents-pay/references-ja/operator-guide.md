# agents-pay オペレーターガイド

オペレーター向けのセットアップとロールのガイダンスです。**デプロイする前に、IAM のセクションをお読みください。**

(スキルレベルの README ではなく `references/` に置いているのは、このリポジトリでは README がプラグインレベルにあり、各スキルのオペレーター向け資料はすべて `references/` に入れるためです。)

すべての支払いの制御は、署名の前に、オペレーターが所有する設定ファイルをもとに、コードの中で強制されます。モデルへの指示は、アクセス制御ではありません。

---

## デプロイする前に: IAM ロールの分離

**これが最も重要なセクションです。最初にお読みください。**

AgentCore Payments は、4 つのロールのモデルを定義しています。公式のガイドに従ってください:
**<https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-iam-roles.html>**

| ロール | 目的 | 使う主体 |
|---|---|---|
| **ControlPlaneRole** | payment manager、connector、認証情報プロバイダー | 管理者 |
| **ManagementRole** | 支払い instrument とセッション。`ProcessPayment` に対する明示的な `Deny` を持つ | **人間**。このスキルの管理用 CLI を実行する |
| **ProcessPaymentRole** | 既存のセッションに対して支払いを実行する。**セッションの書き込みはなし** | **エージェントのランタイム** |
| **ResourceRetrievalRole** | 認証情報を取得するために AgentCore が引き受けるサービスロール | `bedrock-agentcore.amazonaws.com` |

### ルール

> **エージェントは、ManagementRole を持ってはならず、
> `scripts/agents_pay_admin.py` を実行できてもなりません。**

人間が **ManagementRole** を使って、支払い instrument を作成し、各支出セッションを承認します。エージェントは **ProcessPaymentRole** で動作し、すでに承認されたセッションを支出できますが、セッションを作成できません。

エージェントが両方を持つ場合、予算を使い切るたびに、新しい予算を自分で発行できます。そうなると、セッションごとの上限は何も制限しなくなります。AWS のガイドは明確に述べています。

> 「PaymentSession の*書き込み*権限(たとえば `CreatePaymentSession`)と `ProcessPayment` を同じロールに含めないでください。そうしないと、呼び出し元は、より大きな予算で新しいセッションを作成することで、支払いの上限を回避できます。」

そして、そもそも分離が存在する理由について:

> 「支払いの管理と支払いの実行を分離することで、単一の侵害された ID が、無制限の予算でセッションを作成することと、そのセッションに対して支払いを実行することの両方を行うのを防ぎます。」

### 強制する方法

1. **IAM が、本当の境界です。** ランタイムのロールは、`CreatePaymentSession` と、すべての `Create*` のセットアップ用アクションを除外する必要があります。確認:

   ```bash
   aws iam get-role-policy --role-name <ProcessPaymentRole> --policy-name <Policy> \
     | grep -c CreatePaymentSession        # 0 であること
   ```

   ランタイムで `CreatePaymentSession` に対する `AccessDeniedException` が出るのは、**制御が機能している証拠**です。権限を付与して「修正」しないでください。

2. **管理用の CLI を、エージェントの手が届かない場所に置く。** ランタイムのイメージではなく、ワークステーションまたは別のホストに置きます。

3. **TTY のゲートは、多層防御であり、境界ではありません。** `new-session` は、対話型のターミナルがないと実行を拒否し、`--yes` フラグもないため、ヘッドレスのエージェントは操作できません。しかし、対話型のターミナルでユーザーとして動作しているエージェントなら、操作できてしまう可能性があります。それを実際に止めるのは、IAM です。

### どのロールがどのコマンドを実行するか

| ステップ | コマンド | ロール |
|---|---|---|
| 1 | `agentcore add payment-manager` / `payment-connector` / `deploy` | ControlPlaneRole |
| 2 | `agents_pay_admin.py init-config` | なし — ローカルファイルを書き込む |
| 3 | `agents_pay_admin.py create-instrument` | **ManagementRole** |
| 4 | `agents_pay_admin.py new-session` | **ManagementRole** |
| 5 | エージェントが `x402_fetch` を呼び出す | **ProcessPaymentRole** |

---

## クイックスタート

```bash
# 0. 前提条件
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
npm install -g @aws/agentcore

# 1. プロビジョニング（ControlPlaneRole。プロバイダーのシークレットに触れる唯一のステップ）
agentcore add payment-manager                     # フラグなし — 対話式のウィザード
agentcore add payment-connector                   # フラグなし — 対話式のウィザード
agentcore deploy

# 2. 承認された受取人を少なくとも 1 つ指定して、設定を書く
python scripts/agents_pay_admin.py init-config \
  --max-per-payment-usd 0.05 \
  --recipient 0xMerchantWalletAddress

# 3. ウォレットを作成する（ManagementRole）— 支払者のアイデンティティは設定から取得される
python scripts/agents_pay_admin.py create-instrument --email you@example.com

# 3a/3b. ここで、エンドユーザーがブラウザで委任を行い、ウォレットに USDC を入金する。
#        どちらもウォレットに紐づくもので、どちらもステップ 4 の前に行う必要がある —
#        セッションには時間制限があるため、先に作成すると、時間が無駄に過ぎるだけ。
#        ステップ 3 が、委任の URL とウォレットアドレスを表示する。

# 4. 予算を承認する（ManagementRole。対話式 — 「approve」と入力する）
python scripts/agents_pay_admin.py new-session --budget 1.00

# 5. 検証してから、エージェントにツールを登録する
python scripts/agents_pay_admin.py preflight
```

### 対話式の承認を具体的に

ステップ 4 は、入力を求める通常のコマンドです。Web コンソールでもコールバックでもありません。自分のシェルで実行して、1 つの単語を入力します。

```text
$ python3 scripts/agents_pay_admin.py new-session --budget 1.00

About to create a payment session:
  manager   : arn:aws:bedrock-agentcore:us-west-2:1234:payment-manager/pm-abc
  payer     : agents-pay-0e2ed33d1100
  budget    : 1.00 USD (hard cap for the whole session)
  expires in: 60 minutes
Type 'approve' to continue: approve        <-- これを入力する

Payment session created: ps-7f3a9c2e
Recorded in            : ~/.agents-pay/config.json  (nothing to copy by hand)
```

`approve` 以外を入力すると中止され、セッションは作成されません。stdin がターミナルでない場合(エージェントがシェルを呼び出す、CI、パイプ)は、入力を求めずに、コマンドが完全に拒否されます。

```text
Refusing to create a payment session without an interactive terminal.
Session creation requires a human typing 'approve' at a TTY. There is no
non-interactive mode: that would let an automated caller mint spending budget.
```

`approve` をパイプで渡しても、動作しません。

### OpenClaw

エージェントが OpenClaw で動作する場合は、Python の関数を手作業で組み込む代わりに、公開されているプラグインをインストールします。

```bash
openclaw plugins install clawhub:@aws/aws-agents-pay
openclaw plugins inspect aws-agents-pay               # 登録されるものを確認する
```

OpenClaw は、TypeScript プラグインの `get_paid_content` ツールを通じて支払いを実行します。その他のホストは、同等の Python の `x402_fetch` ツールを使います。ランタイムの経路は、ちょうど 1 つ選んでください。両方を有効にすると、ポリシーの強制が経路に依存してしまいます。

OpenClaw のランタイムを信頼する前に、確認すべきことが 2 つあります。

1. **ウォレットのシークレットやプロバイダーのキーを、パラメーターとして受け取るツールがないこと。** 認証情報は、環境または `agentcore` のウィザードに置くべきもので、モデルから見えるスキーマには決して置きません。そうでないと、トランスクリプト、トレース、ログに残ります。
2. **モデルが呼び出せるツールで、支払いセッションを作成するものがないこと。** モデルが予算を作れる場合、セッションごとの上限は何も制限しません。

どちらかが誤っている場合は、Python の `x402_fetch` の経路に切り替える前に、プラグインを無効にしてください。

### 代わりに Strands のプラグインや LangGraph のミドルウェアを使う場合

これらの統合は、フレームワークの内部で支払いを決済するため、**このスキルの制御は一切適用されません** — 1 回あたりの上限も、オリジンの審査も、導出された冪等性トークンもありません。さらに、`auto_session=True` はランタイムのロールに `CreatePaymentSession` を必要とします。これは、IAM のガイドが警告している構成です。

プロトタイプや、ツールの面をすべて自分で所有しているエージェントには、妥当な選択です。オープンなウェブに対して、自分のお金を支出するエージェントには、代わりに `x402_fetch` を登録してください。いずれの場合も、**両方を有効にしないでください** — どちらが特定の `402` を決済するかをモデルが選ぶことになり、ゲートは助言に過ぎなくなります。

完全な比較は、[`security-model.md`](security-model.md) の「The gate only covers what routes through it」を参照してください。

### ツールを登録する

```python
from strands import Agent, tool
from x402_fetch import x402_fetch, payment_session_status
agent = Agent(model=..., tools=[tool(x402_fetch), tool(payment_session_status)])
```

---

## エージェントにできることと、できないこと

| 機能 | 関数 | 支出が発生するか | 上限を引き上げられるか |
|---|---|---|---|
| 支払って取得する | `x402_fetch(url)` | はい。2 つの上限の範囲内で | **いいえ** |
| 残りの予算を確認する | `payment_session_status()` | いいえ — 読み取り専用 | **いいえ** |
| ブラウザのナビゲーションの支払い | `prepare_browser_payment(url)` | はい。2 つの上限の範囲内で | **いいえ** |

完全に手の届かないもの: セッションの作成、インフラのプロビジョニング、プロバイダーの認証情報、署名済みの支払い証明。

---

## 設定

単一のファイル、`~/.agents-pay/config.json` — モード `0600`、`0700` のディレクトリ内で、アトミックに書き込まれます。**シークレットは含まれません**: リソースの識別子と上限だけです。

```json
{
  "resources": {
    "payment_manager_arn": "arn:aws:bedrock-agentcore:...:payment-manager/pm-1",
    "payment_instrument_id": "pi-...",
    "payment_session_id": "ps-...",
    "user_id": "agents-pay-0e2ed33d1100",
    "region": "us-west-2"
  },
  "policy": {
    "max_per_payment_usd": "0.05",
    "allowed_networks": ["eip155:84532"],
    "allowed_assets": { "eip155:84532": ["0x036CbD53842c5426634e7929541eC2318f3dCF7e"] },
    "allowed_schemes": ["exact"]
  }
}
```

`create-instrument` と `new-session` が結果をここに書き込むため、手でコピーする必要はありません。承認されたランタイムはこのファイルを読み、設定の書き込み操作は公開しません。

**シングルテナント。** 1 つのインストールが、1 人の支払者です。`init-config` が `user_id` を一度だけ生成し、以降のすべてのステップが設定からそれを読み取ります。そのため、入力し直す必要はなく、`create-instrument` と `new-session` の間で誤って不一致にすることもありません(不一致だと、instrument を支出できないセッションができます)。これは、ログイン名やホスト名から導出するのではなく、ランダムです。支払い API とウォレットの連携アカウントに渡されるためです。本当に必要な場合は、`--user-id` を渡して上書きできます。マルチテナントのランタイムの分離には、テナントごとに別の OS アカウントや分離したランタイムを使います。`AGENTS_PAY_CONFIG` は、人間が実行する管理コマンドのパスを選択するためだけのものです。

**ランタイムの設定パスは固定されています。** `.agents-pay/config.json` を、オペレーティングシステムのアカウントから解決し、`HOME`、`AGENTS_PAY_CONFIG`、`X402_POLICY_FILE` は無視します。そのファイル内のリソースの識別子は、環境変数によるフォールバックよりも優先されます。ファイルに記載がない場合は、コンテナや Lambda が環境変数で識別子を注入することもできますが、ポリシー自体は、固定されたファイルから取得する必要があります。

ファイルのモードが守るのは、他の OS のプリンシパルからであり、オーナーとしてすでに動作している任意のコードからではありません。制限のない同一ユーザーでのシェルアクセスを許可するハーネスは、署名者と設定を、別の OS アイデンティティ、コンテナ、AWS のロールの境界で分離する必要があります。

### オープンなウェブをブラウジングする

デフォルトでは、エージェントは**公開されている任意の HTTPS サイト**を取得できます。`allowed_origins` は任意です。既知の加盟店の集合に固定したい場合にのみ設定します。支払いは通常、承認された受取人を必要とします: チャレンジの `payTo` が `allowed_recipients` に含まれていなければならず、そうでない場合、信頼できるコードが署名の前に拒否します。

新しい加盟店については、支払いの済んでいない 402 のチャレンジを確認し、`payTo`、金額、アセット、ネットワークをレビューしたうえで、意図した許可リストで `init-config --force --recipient <payTo>` を再実行します。承認は、プロンプトではなく、Python のポリシーの中に保ってください。

明示的なオープン受取人のポリシーを使う場合は、どの `--recipient` フラグも付けずに、`init-config` を `--allow-any-recipient` で実行します。これは、受取人の選択を発行元に委ねます。スキーム、ネットワーク、正確なアセット、オリジン/リソース、1 回あたりの上限、セッションの累積予算は、引き続き強制されます。`allow_any_recipient` と `allowed_recipients` を同時に設定しないでください。信頼できるコードは、そのような曖昧なポリシーを拒否します。

### 2 つの上限

| 上限 | 範囲 | 設定する場所 |
|---|---|---|
| セッションの予算 | **累積** — 人間が再承認するまでの合計 | `new-session` |
| `max_per_payment_usd` | **1 回のトランザクションごと** | ポリシーのセクション |

両方が必要です。セッションの予算だけだと、悪意のある 1 つのチャレンジが、残りの全額を要求し、1 回の支払いで使い切られてしまいます。*(1 回あたりの上限は、サービス側のセッションの予算に含まれるのが望ましいと言えます。AgentCore Payments がそれを提供するまでは、スキルがローカルで強制します。)*

---

## セキュリティモデル

モデルは、脅威モデルの**内側**にあるものとして扱われます。プロンプトインジェクションが成功しても、設計が維持されなければなりません。要約すると:

- **支払いの判断は、署名の前に、設定ファイルをもとに、コードの中で行われます:** スキーム、ネットワーク、正確なアセットのコントラクト、受取人、1 回あたりの上限。
- **審査を通ったチャレンジのエントリだけが、署名者に渡されます。** 発行元の生のレスポンスは転送されません。`resource` オブジェクトは、URL との結び付きの検証(オリジン + パスが、リクエストした URL と一致すること)の後にのみ含まれます。発行元が挿入した任意のフィールドは、取り除かれます。
- **署名済みの証明は、モデルに届きません。** ブラウザのフローでは、モデルは、1 つのオリジンとパスに結び付けられ、90 秒で期限が切れる、不透明な使い捨てのハンドルを受け取ります。
- **取得は堅牢化されています:** HTTPS のみ。ループバック、プライベート、リンクローカル、メタデータ、マルチキャスト、予約済み、CGNAT、IPv4 マップのアドレスは拒否。審査を通った IP を固定して、DNS リバインディングの隙を塞ぐ。リダイレクトには従わない。圧縮は拒否し、本文のサイズを制限する。
- **リトライは冪等です。** トークンはランダムではなく導出され、発行元のノンスを除外します。そのため、レスポンスが失われた後のリトライでも、2 回支払うのではなく、1 つの認可が再送されます。
- **支払い済みのコンテンツは、信頼できない入力です** — デフォルトでは、コンテンツタイプ、バイト数、SHA-256 のハッシュだけが返されます。オペレーターが設定ファイルで `return_body: true` を設定した場合、本文は上限付き(10 KiB)で返され、`"untrusted": true` と印が付けられます。その中の指示は、データであり、決してコマンドではありません。

完全な脅威モデルと、強制の一覧表:
[`security-model.md`](security-model.md)。

---

## ファイル

| パス | 役割 |
|---|---|
| `SKILL.md` | エージェントが読み込むスキル |
| `scripts/x402_policy.py` | 信頼できる判断の起点 |
| `scripts/x402_fetch.py` | 堅牢化された取得、決済、セッションのステータス、ブラウザのハンドル |
| `scripts/agents_pay_admin.py` | **人間が実行する**管理用の CLI — エージェントから遠ざけておく |
| `scripts/test_x402_policy.py` | セキュリティの回帰テスト |
| `scripts/test_portability.py` | ランタイム間の移植性チェック |
| [`security-model.md`](security-model.md) | 脅威モデルと、強制の一覧表 |
| [`setup.md`](setup.md) | プロビジョニングの完全なウォークスルーと IAM ポリシー |
| [`troubleshooting.md`](troubleshooting.md) | 拒否と失敗の診断 |

```bash
python3 scripts/test_x402_policy.py      # セキュリティの回帰テスト
python3 scripts/test_portability.py      # ランタイム間のチェック
```

---

## 運用上の注意

- **テストネットから始める。** デフォルトは Base Sepolia を対象とします。メインネットに移行すると、実際のお金が動きます — 先に、上限と受取人のモードを再確認してください。
- **エージェントが支出する可能性のある金額だけを、ウォレットに入金する。** ウォレットの残高は、他のすべての制御が失敗した場合の、最後の砦です。
- **セッションは短く**(60 分以内)、予算は小さく保つ。
- **ランタイムのプリンシパルによる `CreatePaymentSession` にアラームを設定する。** それは不可能なはずなので、発生した場合は、IAM の分離が崩れたことを意味します。
- CloudTrail が `bedrock-agentcore` の呼び出し、特に `ProcessPayment` を記録していることを確認する。
