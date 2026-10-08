# agents-pay のトラブルシューティング

セットアップ時に作成した仮想環境から、Python のコマンドを実行してください。

```bash
source .venv/bin/activate
```

状況は 2 種類あり、どちらにいるのかが重要です。

- **拒否** — 信頼できるコードが、支払いを断った。制御が機能した。(意図的に)ポリシーを修正するか、拒否を受け入れる。
- **失敗** — 何かが誤って設定されているか、壊れている。組み込みを修正する。

`x402_fetch` は、エージェントのループに例外を投げません。JSON を返します。`refused: true` は、バグではなく、ポリシーによる判断を意味します。

> **これを読むエージェントへ:** 拒否は、回避すべき障害ではありません。変更せずにリトライしたり、別のツールで URL を取得したり、続行するために上限を引き上げるようユーザーに迫ったりしないでください。理由を報告して、止まってください。

## 拒否

### "No payment option in this challenge satisfies the configured policy"

包括的な拒否です。意図的に画一的になっています。失敗したフィールドの名前を示すと、悪意のある発行元が、メッセージが変わるまでチャレンジを繰り返し、ポリシーを探り出せてしまいます。代わりに、ローカルで診断してください。

```bash
# 1. 加盟店は、実際に何を求めているか
curl -sD - https://<host>/<path> -o /dev/null \
  | awk '/^payment-required:/{print $2}' | base64 -d | python3 -m json.tool

# 2. ポリシーは何を許可しているか
python3 scripts/agents_pay_admin.py show-config
```

次の順に比較します。

| チェック | チャレンジのフィールド | ポリシーのキー |
|---|---|---|
| 受取人 | `accepts[].payTo` | `allowed_recipients` |
| 金額 | `accepts[].amount`(基本単位) | `max_per_payment_usd` |
| ネットワーク | `accepts[].network` | `allowed_networks` |
| アセット | `accepts[].asset` | `allowed_assets[network]` |
| スキーム | `accepts[].scheme` | `allowed_schemes` |

金額は整数の基本単位で、USDC は小数点以下 6 桁です: `2000` = $0.002、`500000` = $0.50。上限が `0.001` なら、`2000` のチャレンジは拒否されます — これは正しい計算であり、バグではありません。

どのルールが発動したかを確認するには、実際のチャレンジを直接評価します。

```bash
python3 - <<'PY'
import base64, json, httpx, sys
sys.path.insert(0, "scripts")
import x402_policy as pol
url = "https://<host>/<path>"
r = httpx.get(url, timeout=15)
ch = json.loads(base64.b64decode(r.headers["payment-required"]))
print(json.dumps(ch["accepts"], indent=2))
try:
    print("AUTHORIZED:", pol.authorize_payment(url, ch))
except pol.PolicyError as e:
    print("REFUSED:", e)
PY
```

加盟店の値が本当に想定されたものである場合にだけ、ポリシーを緩めてください。修正した許可リストで `init-config --force` を再実行します — モードや所有者を手で編集しないでください。

### "Origin ... is not in the configured allowed_origins"

ホストが許可リストにありません。`--origin https://host` で追加します(スキーム + ホスト、パスなし、末尾のスラッシュなし)。デフォルトの HTTPS ポートは正規化されて取り除かれるため、`https://host:443` と `https://host` は同じオリジンです。

### "Only https:// payment URLs are allowed"

プレーンな HTTP、`file://`、その他のスキームは拒否されます。支払いのトラフィックは暗号化されている必要があり、上書きする方法はありません。

### "Payment URL resolves to a non-public address"

ホスト名が、ループバック、RFC1918、リンクローカル、メタデータ(169.254.x.x)、マルチキャスト、予約済み、未指定、CGNAT の空間に解決されました。これは SSRF のガードです。正当な加盟店がこれに引っかかる場合、その加盟店はインターネットから到達できず、このエージェントからの有効な支払い先ではありません。

### "No payment policy at ..."

許可的なデフォルトはありません — ポリシーが存在するまで、支払いは拒否されます。

```bash
python scripts/agents_pay_admin.py init-config \
  --max-per-payment-usd 0.05 \
  --recipient 0xMerchantWalletAddress
```

### "... has mode -rw-rw-r--; expected 0600" / "is a symlink"

ポリシーファイルは、セキュリティ制御です。そのため、他者が書き込めるファイルやシンボリックリンクは、ポリシーがないものとして扱われます。設計上、安全側に倒れて失敗します。

```bash
chmod 600 ~/.agents-pay/config.json
```

シンボリックリンクの場合は、実ファイルに置き換えてください。自分が所有していない場合は、「修正」する前に調べてください。誰かが別のユーザーとして書き込んだ可能性があります。

### "Payment policy allows no recipients" / 受取人のモードは排他的

空の許可リストは、すべてを拒否します。キーがない場合は、許可ではなく拒否です。暗黙のワイルドカードはありません。少なくとも 1 つの `--recipient` を付けて `init-config` を再実行するか、代わりに `--allow-any-recipient` を意図的に使ってください。2 つのモードを組み合わせてはいけません。加盟店のホストの集合に固定したい場合にだけ、`--origin` を追加します。

## 失敗

### "PAYMENT_MANAGER_ARN is not set"(または INSTRUMENT/SESSION/USER)

ランタイムの識別子が不足しています。どれか確認するには、`preflight` を実行します。

```bash
python3 scripts/agents_pay_admin.py preflight
```

manager の ARN と connector の ID は、`agentcore/.cli/deployed-state.json` から、セッション ID は `new-session` から取得されます。

### "bedrock-agentcore with payments support is not installed"

```bash
python -m pip install --upgrade 'bedrock-agentcore>=1.19.0'
python -c "from bedrock_agentcore.payments import PaymentManager; print('OK')"
```

### "Response exceeded N bytes"

本文が `X402_MAX_BODY_BYTES`(デフォルトは 256 KiB)を超えたため、それ以上はバッファリングされませんでした — メモリ枯渇への対策です。加盟店が正当に大きい場合は、意図的に引き上げてください。

```bash
export X402_MAX_BODY_BYTES=1048576
```

### コンテンツが返されない: `body_returned: false`

デフォルトでは、支払い済みのレスポンスの本文は、モデルのコンテキストに返されません。レスポンスには、コンテンツタイプ、バイト数、SHA-256 のハッシュだけが含まれます。本文の返却を有効にするには、`~/.agents-pay/config.json` のポリシーのセクションで、`return_body: true` を設定します。有効にすると、コンテンツは 10 KiB に制限され、`untrusted: true` と印が付けられます。

より大きい、またはよりリッチなコンテンツ(HTML など)を処理する必要がある場合は、支払いツールを持たない別のコンテキストで、取得して要約してください。

### 支払いのフロー中の `ConnectError`

通常は、支払いの経路ではなく、TLS の信頼の問題です。確認:

```bash
python3 -c "import ssl; print(ssl.get_default_verify_paths().cafile)"
python3 -c "import certifi; print(certifi.where())"
```

`x402_fetch` は、certifi の CA バンドルがあればそれを優先します。一部のインタープリター(特に mise が管理する Python)には、システムの CA ファイルがないためです。検証が無効になることはありません — 回避するのではなく、`certifi` をインストールしてください。

### コンテンツではなく 30x のレスポンス

リダイレクトには決して従いません。402 の後のリダイレクトは、署名済みの証明を別のオリジンに移動させてしまう可能性があるためです。代わりに、エージェントを最終的な URL に向けてください。

### サービスからの ProcessPayment のエラー

| メッセージ | 意味 | 対応 |
|---|---|---|
| `Payment session not found` | セッションが無効、または削除された | 人間が `new-session` を実行する |
| `PaymentSessionExpired` | `expiry_time_in_minutes` を過ぎた | 人間が `new-session` を実行する |
| `Payment instrument not found` / `does not belong to user` | instrument とユーザーの不一致 | `PAYMENT_USER_ID` が instrument のユーザーと一致することを確認する |
| `Payment connector is not active` | まだプロビジョニング中、または削除された | 状態を確認する。必要なら再作成する |
| `Network mismatch` | チャレンジのチェーン ≠ instrument のファミリー | `ETHEREUM` は Base/Base Sepolia を、`SOLANA` は Solana をカバーする |
| `Wallet does not have a USDC balance` | ウォレットに入金されていない | <https://faucet.circle.com/> から入金する |
| `Failed to obtain resource payment token` | サービスロールに、トークンボールト/シークレットへのアクセス権がない | ロールを修正する。IAM の反映のため約 15 秒待つ |
| `Failed to assume payment execution role` | 信頼ポリシーが誤っている | 正しい `aws:SourceAccount` を付けて、`bedrock-agentcore.amazonaws.com` を信頼する必要がある |
| `Delegated signing grant is not active` | エンドユーザーが委任していない | `redirectUrl`(Coinbase)または Privy SDK のフローを完了する |
| `Delegated signing is not enabled` | CDP のプロジェクト設定がオフ | Portal → Wallet → Embedded Wallets → Policies → 有効にする |
| `CreatePaymentSession` での `AccessDeniedException` | ランタイムがセッションを発行しようとした | **想定どおりで、正しい動作。** セッションは人間のみ |

### ポリシーが拒否するはずの支払いが通ってしまった

ほぼ常に、`x402_fetch` 以外の何かが決済したことが原因です。ゲートが経路に入るのは、このスキルのツールを経由する呼び出しだけです。

フレームワークネイティブな統合が有効でないか確認します。

```bash
grep -rn 'AgentCorePaymentsPlugin\|AgentCorePaymentsMiddleware\|payments.integrations' .
```

どちらかが登録されている場合、支払いはフレームワークの内部で行われ、`x402_policy.py` が参照されることはありません — 上限も、オリジンのチェックも、導出されたトークンもありません。それを削除して `x402_fetch` を登録するか、制御が適用されないことを受け入れてください。両方を動かすと、どちらが特定の `402` を決済するかをモデルが選びます。

また、`auto_session=True` はセッションの書き込み権限を必要とするため、ランタイムのロールに本当にその権限がないことも確認してください。

```bash
aws iam get-role-policy --role-name <ProcessPaymentRole> --policy-name <Policy> \
  | grep -c CreatePaymentSession        # 0 であること
```

### プロバイダー固有のエラー

#### Coinbase CDP

| メッセージ | 原因 | 対処 |
|---|---|---|
| `Delegated signing is not enabled` | プロジェクト設定がオフ | portal.cdp.coinbase.com → Project → Wallet → Embedded Wallets → Policies → Delegated signing を有効にする |
| `Delegated signing grant is not active` / `Delegation not completed` | エンドユーザーが委任していない | `create-instrument` の `redirectUrl` にアクセスし、サインインして、ウォレットアドレスへのアクセスを付与してもらう |

#### Stripe Privy

| メッセージ | 原因 | 対処 |
|---|---|---|
| `Privy credentials are invalid` | 認証情報プロバイダーの App ID または App Secret が誤っている | Privy のダッシュボードで両方を再確認し、connector を再作成する |
| `Privy appId is invalid or missing` | 認証情報プロバイダーの `appId` が誤っている | ダッシュボードで修正し、再作成する |
| `Privy signing key is invalid or expired` | Authorization キーがローテーションされた、または期限切れ | 新しい P-256 の鍵ペアを生成する(Wallet Infrastructure → Authorization)。**`wallet-auth:` プレフィックスを削除し**、生の base64 を残す |
| `Wallet policy denied the transaction` | Privy のウォレットポリシーがブロックしている | ダッシュボードで、金額、受取人、頻度の上限を確認する |
| `The linked account data is invalid` | `linkedAccounts` のメールアドレスの形式が不正 | 有効なアドレスで instrument を再作成する |
| `Rate limited by Privy` | Privy API のスロットリング | 間隔をあけてリトライする |
| `Delegation not completed` | エージェントのキーが、ウォレットの署名者として追加されていない | [Privy のフロントエンド SDK](https://github.com/privy-io/aws-agentcore-sdk)を使い、エンドユーザーのメールアドレスでサインインして、委任を承認する |

Privy には `redirectUrl` がありません。委任は、ホスト型のページではなく、フロントエンド SDK を通じて行われます。`create-instrument` が委任の URL を表示しなかった場合、Privy では想定どおりで、失敗ではありません。

最後の行は、改めて述べる価値があります。`CreatePaymentSession` での、ランタイムの `AccessDeniedException` は、設定ミスではなく、制御が機能している証拠です。権限を付与して「修正」しないでください。

### メソッドが許可されていない

`GET` と `HEAD` のみです。本文を伴うメソッド(`POST`、`PUT`、`PATCH`)は拒否されます。ポリシーゲートが検証するのは URL であって、リクエストの本文ではないため、許可すると、エージェントが選んだデータを任意のオリジンに送れてしまうからです。有料の API が本当に `POST` を必要とする場合、その呼び出しは、このツールの背後ではなく、自分が制御するコードの中に置くべきです。

### 支払い済みなのに、加盟店がまだ 402 を返す

テストネットの決済は断続的に遅いため、ツールはすでに、最大 `X402_MAX_PAYMENT_ATTEMPTS` 回(デフォルトは 5、1〜10 に制限)、これを自動でリプレイしています。`"attempts": 5` と `"paid": false` が表示された場合、5 回のリプレイがすべて 402 を受け取りました。毎回同じ認可がリプレイされるため、支払いは失われておらず、二重にもなっていません。少し待ってからリトライするか、上限を引き上げてください。

```bash
export X402_MAX_PAYMENT_ATTEMPTS=8
```

リトライは、導出された同じ `client_token` を再利用するため、再試行しても、2 回支払うのではなく、*同じ*認可がリプレイされます。次のコマンドで確認できます。

```bash
python3 scripts/test_x402_policy.py -k IdempotencyTests
```

解消しない場合は、加盟店の決済が遅れている(テストネットでよくある)か、x402 のバージョンの扱いが異なっています。チャレンジの `x402Version` を確認して、少し待ってからリトライしてください。「ただ取得する」ためにツールを迂回しないでください — それが、二重支払いが起きる原因です。

## ヘルスチェック

```bash
python3 scripts/test_x402_policy.py                  # すべて合格すること
python3 scripts/agents_pay_admin.py show-config      # 0600、自分が所有している
python3 scripts/agents_pay_admin.py preflight        # 組み込み + 余分なシークレットがないこと
aws iam get-role-policy --role-name <RuntimeRole> --policy-name <P> \
  | grep -c CreatePaymentSession                     # 0 であること
```
