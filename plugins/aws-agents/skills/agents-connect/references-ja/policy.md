# policy

AgentCore のエージェントが実行できることを制御します。ツール呼び出しの制限、ビジネスルールの適用、機微な操作の保護を行います。

## 使うタイミング

- エージェントが呼び出せるツールを制限したい
- ビジネスルールを適用したい（例: 返金は 500 ドル未満のみ）
- エージェントのアクションに、ロールベースのアクセス制御をかけたい
- 特定のツールに対して、緊急停止のキルスイッチが欲しい
- ポリシーが、許可を期待するリクエストを拒否している（または、拒否を期待するものを許可している）

## 入力

`$ARGUMENTS` は省略可能です。

```
/policy                     # 対話形式 — 何を制限したいかを質問します
/policy generate            # 自然言語から Cedar を生成します
/policy debug               # ポリシーが許可/拒否している理由を診断します
/policy emergency           # 緊急停止用のポリシーを生成します
```

## AgentCore のポリシーの仕組み

AgentCore Policy は、**Gateway の境界**で、Cedar ベースの認可ルールを適用します。ツール呼び出しがターゲットに到達する前に評価されます。すべてのツール呼び出しが、リアルタイムでポリシーに照らして評価されます。

**デフォルトの動作:** Gateway にポリシーエンジンがアタッチされていない場合、すべてのツール呼び出しが許可されます。ポリシーエンジンをアタッチすると、デフォルトは**拒否**になります。許可したいすべてのものに対して、明示的な `permit` ポリシーを書く必要があります。

**主な概念:**

- **ポリシーエンジン** — ポリシーを入れるコンテナ。Gateway にアタッチされる
- **ポリシー** — 特定のアクションを許可または禁止する Cedar のルール
- **`forbid` は `permit` に優先する** — いずれかの forbid ポリシーに一致すると、permit ポリシーに関係なく、そのアクションは拒否される

---

## 手順

### ステップ 1: プロジェクトを読み取る

`agentcore/agentcore.json` を読み、次のことを把握します。

- どの Gateway があるか（`agentCoreGateways` 配列）
- ポリシーエンジンがすでに設定されているか（`policyEngines` 配列）

### ステップ 2: 目的を理解する

質問します（または `$ARGUMENTS` から推測します）。

> 「何を制御したいですか?
>
> 1. 入力値に基づいてツールを制限する（例: 金額 < 500 ドル）
> 2. ロールベースのアクセス（特定のユーザーだけが、特定のツールを呼び出せる）
> 3. 特定のツールを完全にブロックする
> 4. 緊急停止 — すべてのツールを直ちに無効にする
> 5. ポリシーが予期せず許可または拒否している理由をデバッグする」

---

## Path A: ポリシーエンジンをセットアップする

### ステップ A1: ポリシーエンジンを作成する

```bash
# 作成して、既存の Gateway にアタッチする
agentcore add policy-engine \
  --name MyPolicyEngine \
  --attach-to-gateways MyGateway \
  --attach-mode LOG_ONLY
```

**まず `LOG_ONLY` モードで始めてください。** ポリシーは評価されてログに記録されますが、強制はされません。これにより、強制を有効にする前に、ポリシーが正しく動作することを確認できます。

準備ができたら、`ENFORCE` に切り替えます。

```bash
# 既存の Gateway を更新する
agentcore add gateway \
  --name MyGateway \
  --policy-engine MyPolicyEngine \
  --policy-engine-mode ENFORCE
```

（同じ `--policy-engine` と `--policy-engine-mode` のフラグは、Gateway の作成時にも使えます。）

### ステップ A2: デプロイして有効にする

```bash
agentcore deploy -y
```

---

## Path B: Cedar ポリシーを書く

> [!WARNING]
> `resource` フィールドで特定の Gateway の ARN を参照する Cedar ポリシーは、先に Gateway を
> デプロイしておく必要があります。Gateway が AWS 上に存在する前に、Gateway の ARN を
> 含むポリシーを追加することはできません。
>
> 2 段階のデプロイ:
>
> 1. 先に Gateway をデプロイする: `agentcore deploy -y`
> 2. Gateway の ARN を取得する: `agentcore status --type gateway --json`
> 3. 実際の ARN でポリシーを追加し、もう一度デプロイする
>
> `-g` / `--generate` フラグもデプロイ済みの Gateway を必要とします。自然言語を Cedar に
> 変換するために、Gateway の ARN を必要とする AWS の API を呼び出すためです。
> Gateway をデプロイする前に `-g` を実行すると、失敗します。

### オプション 1: 自然言語による生成（最も簡単）

**先に Gateway をデプロイしておく必要があります。** CLI は、Gateway の ARN を必要とする API を呼び出すためです。

```bash
# 先に Gateway をデプロイする
agentcore deploy -y

# その後、ポリシーを生成する（--gateway は、どのデプロイ済み Gateway を使うかを CLI に伝える）
agentcore add policy \
  --name refund_policy \
  --engine MyPolicyEngine \
  -g "Allow users with the refund-agent role to process refunds when the amount is less than 500" \
  --gateway MyGateway
```

CLI は、説明から Cedar を生成し、Gateway の ARN を自動で解決して、結果を検証します。デプロイする前に、生成されたポリシーを確認してください。

**ポリシー名のルール:** 英字、数字、アンダースコアのみ — **ハイフンは使えません**。`refund-policy` は失敗し、`refund_policy` は動作します。

### オプション 2: Cedar を直接書く

`.cedar` ファイルに保存して、登録します。ポリシーが `resource` フィールドで Gateway の ARN を参照する場合は、事前のデプロイで得た ARN が必要です。

```bash
# デプロイ後に Gateway の ARN を取得する
agentcore status --type gateway --json | jq -r '.gateways[0].arn'

# .cedar ファイルを実際の ARN で更新してから、ポリシーを追加する
agentcore add policy \
  --name refund_policy \
  --engine MyPolicyEngine \
  --source policy.cedar
```

### Cedar の構文リファレンス

**アクション名の形式:** `AgentCore::Action::"TargetName___tool_name"` — ターゲット名とツール名の間に、アンダースコアが 3 つ入ります。これは、Cedar で最もよくある間違いです。

```cedar
// TargetName は、Gateway ターゲットの名前（agentcore add gateway-target --name で指定したもの）
// tool_name は、そのターゲット内のツール名
AgentCore::Action::"RefundTarget___process_refund"
//                              ^^^
//                         アンダースコアが 3 つ
```

**プリンシパルの種類:**

- `AgentCore::OAuthUser` — OAuth/JWT で認証されたユーザー
- `AgentCore::IamEntity` — IAM で認証された呼び出し元（Gateway が AWS_IAM 認証を使う場合）。`id` 属性に、IAM の完全な ARN が入る

**リソースの形式:**

```cedar
AgentCore::Gateway::"arn:aws:bedrock-agentcore:<REGION>:<YOUR_ACCOUNT_ID>:gateway/<GATEWAY_ID>"
```

Gateway の ARN の取得方法: `agentcore status --type gateway --json | jq -r '.gateways[0].arn'`

### よくあるポリシーのパターン

**金額ベースの制限:**

```cedar
permit(
  principal is AgentCore::OAuthUser,
  action == AgentCore::Action::"RefundTarget___process_refund",
  resource == AgentCore::Gateway::"arn:aws:bedrock-agentcore:us-east-1:123456789012:gateway/my-gateway-id"
)
when {
  principal.hasTag("role") &&
  principal.getTag("role") == "refund-agent" &&
  context.input.amount < 500
};
```

**ロールベースのアクセス（OAuth ユーザー）:**

```cedar
permit(
  principal is AgentCore::OAuthUser,
  action == AgentCore::Action::"AdminTarget___delete_record",
  resource == AgentCore::Gateway::"arn:..."
)
when {
  principal.hasTag("role") &&
  ["admin", "superuser"].contains(principal.getTag("role"))
};
```

**アカウントベースのアクセス（IAM エンティティ）:**

```cedar
permit(
  principal is AgentCore::IamEntity,
  action == AgentCore::Action::"AdminTarget___delete_record",
  resource == AgentCore::Gateway::"arn:..."
)
when {
  principal.id like "arn:aws:iam::123456789012:*"
};
```

**特定のツールを完全にブロックする:**

```cedar
forbid(
  principal,
  action == AgentCore::Action::"PaymentTarget___transfer_funds",
  resource == AgentCore::Gateway::"arn:..."
);
```

**緊急停止 — すべてのツールを無効にする:**

```cedar
forbid(principal, action, resource);
```

**必須フィールドの検証:**

```cedar
forbid(
  principal is AgentCore::OAuthUser,
  action == AgentCore::Action::"InsuranceTarget___file_claim",
  resource == AgentCore::Gateway::"arn:..."
)
unless {
  context.input has description &&
  context.input has priority
};
```

### Cedar の重要なルール

**`getTag()` の前に、必ず `hasTag()` を使う:**

```cedar
// ❌ 誤り — タグが存在しないとエラーになる
when { principal.getTag("role") == "admin" }

// ✅ 正しい — 先に存在を確認する
when {
  principal.hasTag("role") &&
  principal.getTag("role") == "admin"
}
```

**デフォルトは拒否:** ポリシーエンジンが ENFORCE モードでアタッチされると、一致する `permit` ポリシーがない限り、すべてが拒否されます。許可したいすべてのアクションに対して、明示的な permit を書いてください。

**`forbid` は常に優先される:** `forbid` ポリシーは、あらゆる `permit` ポリシーに優先します。緊急停止や、厳格なブロックに使ってください。

---

## Path C: 強制する前にポリシーをテストする

### LOG_ONLY モード

LOG_ONLY モードでは、すべてのリクエストが許可されますが、ポリシーの判定は CloudWatch に記録されます。ENFORCE に切り替える前に、ポリシーを検証するために使います。

```bash
# ポリシーの判定ログを確認する
agentcore logs --runtime MyAgent --since 1h --query "policy"
```

ツール呼び出しごとの `ALLOW` または `DENY` の判定を示すログのエントリを探してください。

### ポリシーの構文を検証する

```bash
agentcore add policy \
  --name test_policy \
  --engine MyPolicyEngine \
  --source policy.cedar \
  --validation-mode FAIL_ON_ANY_FINDINGS
```

Cedar の構文が無効な場合、CLI は、ポリシーを作成する前に検証エラーを返します。

### ENFORCE に切り替える

LOG_ONLY の結果が正しいと確認できたら:

```bash
# Gateway を更新して、強制モードにする
agentcore add gateway \
  --name MyGateway \
  --policy-engine MyPolicyEngine \
  --policy-engine-mode ENFORCE
agentcore deploy -y
```

---

## Path D: ポリシーの失敗をデバッグする

**許可を期待するツール呼び出しで "Access denied" が出る:**

1. このアクションに対する `permit` ポリシーが存在することを確認する — デフォルトは拒否であることを思い出してください
2. アクション名の形式を確認する: `TargetName___tool_name`（アンダースコアが 3 つ）
3. リソースの ARN が、Gateway の実際の ARN と一致していることを確認する
4. 条件内で、`getTag()` の前に `hasTag()` が使われていることを確認する
5. LOG_ONLY のログで、ポリシーエンジンが何を評価しているかを確認する

```bash
# 最近のポリシー判定を確認する
agentcore logs --runtime MyAgent --since 1h --query "policy"
agentcore status --type policy-engine
```

**ポリシーエンジンをアタッチしたら、"すべてが拒否される":**
ポリシーエンジンをアタッチしたものの、`permit` ポリシーをまだ書いていません。デフォルトは拒否です。許可したいアクションに対して、少なくとも 1 つの `permit` ポリシーを書いてください。

**ポリシー名の検証エラー:**
ポリシー名は `^[A-Za-z][A-Za-z0-9_]*$` に一致する必要があります — 英字、数字、アンダースコアのみで、先頭は英字です。ハイフンは使えません。

---

## 出力

- ポリシーエンジンとポリシーを作成するための CLI コマンド
- 要求されたユースケース向けの Cedar ポリシーのファイル
- 強制の前に行う、LOG_ONLY によるテストのワークフロー
- ポリシーの失敗に関するデバッグのガイダンス
