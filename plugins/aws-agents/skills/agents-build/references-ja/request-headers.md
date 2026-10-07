# request-headers

呼び出し元からのカスタム HTTP ヘッダーを、エージェントの呼び出しコードに渡します。

## 使うタイミング

- アプリからエージェントに、テナント ID、相関 ID、フィーチャーフラグを渡す必要がある
- 特定のヘッダーを必要とするプロトコル（A2A、MCP、ベンダー固有のもの）を実装している
- OpenTelemetry の baggage やトレースヘッダーを、呼び出し元から伝播させたい
- リクエストにヘッダーを追加したのに、エージェントのコードから見えない
- 冪等性キーなどのヘッダーを使う外部システムと連携している

## デフォルト: ほとんどのヘッダーは削除される

AgentCore Runtime は、リクエストがエージェントのコードに届く前に、受信したすべてのヘッダーを削除します。**ただし次のものを除きます。**

- `Authorization` — 常に渡されます
- `X-Amzn-Bedrock-AgentCore-Runtime-Custom-*` に一致するヘッダー — これはカスタムヘッダー用の予約済みプレフィックスです

それ以外のもの（`X-Tenant-Id`、`X-Correlation-Id`、`traceparent`、`A2A-Version`、`Idempotency-Key` など）は、ランタイムのリクエストヘッダー許可リストに明示的に追加しない限り、呼び出しコンテキストに現れません。

これは意図的なセキュリティ上の境界です。ランタイムは、呼び出し元が指定した任意のヘッダーをデフォルトでは転送しません。また、これは「送っているヘッダーがエージェントから見えないのはなぜか」という、開発者からの質問で最も多い原因でもあります。

## カスタムデータを渡す 2 つの方法

### 方法 1: 予約済みプレフィックスを使う

呼び出し元で、ヘッダーの名前を `X-Amzn-Bedrock-AgentCore-Runtime-Custom-` プレフィックスを使うものに変更します。ランタイムの設定を変更しなくても、そのまま渡されます。

```
# 呼び出し元が送信する:
X-Amzn-Bedrock-AgentCore-Runtime-Custom-Tenant-Id: acme-corp
X-Amzn-Bedrock-AgentCore-Runtime-Custom-Correlation-Id: 8b2e3d...

# エージェントのコードは、呼び出しコンテキストで同じヘッダーを見る
```

アプリとエージェントの両方を自分で管理しているヘッダーには、これが最も簡単な方法です。

### 方法 2: ヘッダーをリクエストヘッダーの許可リストに追加する

ヘッダー名がプロトコルや外部システムによって決まっている場合（A2A は `A2A-Version` と `A2A-Extensions` を必要とし、OpenTelemetry は `traceparent` と `baggage` を使い、一部の API は `Idempotency-Key` を使います）、名前を変更できません。ランタイムで明示的に許可するよう設定します。

**`agentcore/agentcore.json` を編集**し、ランタイムのエントリに `requestHeaderAllowlist` を追加します。

```json
{
  "runtimes": [
    {
      "name": "MyAgent",
      "requestHeaderAllowlist": [
        "X-Amzn-Bedrock-AgentCore-Runtime-Custom-X-Tenant-Id",
        "X-Amzn-Bedrock-AgentCore-Runtime-Custom-A2A-Version"
      ]
    }
  ]
}
```

その後、`agentcore deploy` を実行します。ファイルの先頭にある `$schema` の URL（`https://schema.agentcore.aws.dev/v1/agentcore.json`）により、すべてのフィールドで IDE のオートコンプリートと検証が使えます。

**CLI のショートカット** — `agentcore add agent --request-header-allowlist "X-Tenant-Id,A2A-Version"` でも、同じ配列が書き込まれます。**重要:** CLI は、`agentcore.json` に書き込むときに、各エントリへ `X-Amzn-Bedrock-AgentCore-Runtime-Custom-` を自動で付与します。JSON を手で編集する場合は、プレフィックス付きの形式を直接書いてください。CLI を使う場合は、短い名前を渡して、プレフィックスの付与は CLI に任せてください。

`Authorization` はデフォルトで渡されるため、許可リストに入れる必要はありません。

### 制約

- 許可リストは**最大 20 ヘッダー**（`Authorization` を明示的に含める場合は、それも数えます）
- **ヘッダー名の長さ:** 256 文字まで
- **ヘッダー値のサイズ:** 1 ヘッダーあたり 4 KB まで
- **名前は大文字小文字を区別します** — 送信される形のとおりに記載してください
- **変更は、ランタイムの次回デプロイ後に反映されます**

20 ヘッダーの上限に達した場合は、多数のヘッダーに分けるのではなく、関連するデータを JSON エンコードした 1 つのヘッダーにまとめてください。

## よくあるユースケース

### マルチテナンシー

```
呼び出し元:         X-Tenant-Id: acme-corp
エージェントのコード: ヘッダーからテナントを読み取り、Memory・データ・ツールをテナントごとにスコープする
```

`X-Tenant-Id` を許可リストに追加します。これにより、エージェントは Memory の名前空間、データベースクエリ、ツール呼び出しの認可を、テナントごとに分離できます。

### 分散トレーシングの伝播

```
呼び出し元:         traceparent: 00-<trace-id>-<span-id>-01
                    baggage:     userId=alice,env=prod
エージェントのコード: OTel SDK を使って親トレースを継続する
```

`traceparent` と `baggage` を許可リストに追加します。OTel SDK のインストルメンテーションがこれらを自動的に拾い、呼び出し元のトレースにつながるスパンを生成します。

### A2A プロトコルへの準拠

```
呼び出し元:         A2A-Version:    1.0
                    A2A-Extensions: x-capability-foo
エージェントのコード: プロトコルのバージョンに応じて動作を分岐する
```

A2A v1.0 は、これらのヘッダーを必要とします。両方を許可リストに追加してください。A2A v0.3 では、どちらも必要ありません。

### 冪等性キー

```
呼び出し元:         Idempotency-Key: 7f3a...
エージェントのコード: キーに基づいて重複排除またはキャッシュを行う
```

冪等性のある外部 API を呼び出すエージェントでは、呼び出し元のキーをエージェントのアウトバウンド呼び出しまで引き継ぐことで、リトライ時の副作用の重複を避けられます。

## エージェントのコードでヘッダーを読む

ヘッダーは、呼び出しハンドラーに渡されるランタイムの `context` オブジェクトに入って届きます。正確なアクセサーはフレームワークによって異なるため、使用する言語の bedrock-agentcore SDK のドキュメントを確認してください。Python の場合:

```python
@app.entrypoint
def invoke(payload, context):
    tenant = context.headers.get("X-Tenant-Id")
    correlation_id = context.headers.get("X-Correlation-Id")
    # ... 必要に応じて使う
```

許可リストにないヘッダーは、コンテキストに存在しません（空文字列ではなく、存在しない状態です）。

## うまくいかないケース

- **許可リストを設定せずにヘッダーを送る** — デフォルトで渡されるもの以外は、通知なしに破棄されます。エージェントのコードにはヘッダーが見えず、エラーも出ません。届くはずのヘッダーが届かない場合は、ランタイムの `requestHeaderConfiguration` を確認してください。
- **シークレットに使う** — 4 KB の値と許可リストの設定は、メタデータ用に設計されており、認証情報用ではありません。API キー、OAuth トークン、シークレットには、AgentCore Identity の認証情報プロバイダーを使用してください。`agents-connect` のパス D を参照してください。
- **動的なヘッダー** — 許可リストは静的なランタイム設定です。リクエストごとに変えることはできません。

## トラブルシューティング

**"My agent doesn't see the header I'm sending"**
次の順に確認してください。(1) ヘッダーが許可リストに入っているか。(2) 大文字小文字を含めて、綴りが完全に一致しているか。(3) 許可リストを更新したあと、ランタイムを再デプロイしたか。(4) 呼び出し元が実際にヘッダーを送信しているか（`curl -v` などのネットワーク検査で確認します）。

**"I hit the 20-header limit"**
関連するデータを、JSON エンコードした 1 つのヘッダーにまとめます。たとえば、`X-Region`、`X-Environment`、`X-Service-Name` の 3 つのヘッダーに分ける代わりに、`X-Context: {"region":"us-west-2","env":"prod","service":"billing"}` を使います。

**"Allowlist update didn't take effect"**
ランタイムを再デプロイしてください。ヘッダーの許可リストは、`agentcore.json` を編集した直後ではなく、次回の `agentcore deploy` で適用される設定です。

## 出力

- プレフィックス方式と許可リスト方式のどちらを使うかの判断
- 必要な場合は、許可リストを更新するための CLI コマンド
- ヘッダーを読み取るためのエージェントのコードパターン
