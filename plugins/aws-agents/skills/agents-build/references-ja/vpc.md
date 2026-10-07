# vpc

AgentCore のエージェントを、VPC 内のプライベートな AWS リソースに接続できるよう設定します。

## 使うタイミング

- エージェントが RDS データベースに接続する必要がある
- エージェントが、インターネットに公開されていない内部 API を呼び出す必要がある
- エージェントのネットワークトラフィックをプライベートに保ちたい
- VPC 接続を設定したのに、接続がタイムアウトする

## 入力

`$ARGUMENTS` は省略可能です。

```
/vpc                        # 対話形式 — 何に接続するかを質問します
/vpc rds                    # RDS データベースへの接続
/vpc debug                  # VPC 接続の問題を診断します
```

## AgentCore の VPC 接続の仕組み

VPC モードを設定すると、AgentCore は VPC のサブネットに **Elastic Network Interface（ENI）** を作成します。この ENI によって、エージェントは VPC 内のプライベート IP アドレスを持ち、プライベートなリソースに到達できます。

**重要なポイント:**

- VPC 接続が直接影響するのは**アウトバウンドのトラフィック**です。ENI は、エージェントのアウトバウンド呼び出しを VPC 経由でルーティングします。**インバウンドのトラフィック**については、AgentCore の VPC エンドポイントを追加することで、PrivateLink 経由で API 呼び出しをプライベートに保てます（これは `networkMode` の設定とは別です）。
- AgentCore は、サービスリンクロール `AWSServiceRoleForBedrockAgentCoreNetwork` を使って ENI を作成します（最初の VPC デプロイ時に自動で作成されます）
- サブネットは**サポートされているアベイラビリティゾーン**にある必要があります。すべての AZ がサポートされているわけではありません。AgentCore が新しいリージョンに展開されるのに伴って、サポートされる AZ の一覧は変わります。

---

## ステップ 0: CLI のバージョンを確認する

`agentcore --version` を実行します。このスキルには v0.9.0 以降が必要です。それより古い場合は、先に進む前に `agentcore update` を実行するよう、開発者に伝えてください。

---

## ステップ 1: サブネットがサポートされている AZ にあることを確認する

AgentCore は、リージョンごとに特定のアベイラビリティゾーン ID のみをサポートしています。サポートされる AZ の一覧は、AgentCore の展開に伴って変わるため、**最新の表は必ず最新のドキュメントで確認してください。**

サブネットの AZ ID を確認します。

```bash
# サブネットの AZ ID を確認する
aws ec2 describe-subnets \
  --subnet-ids subnet-12345678 \
  --query 'Subnets[0].{AZ:AvailabilityZone,AZId:AvailabilityZoneId,SubnetId:SubnetId}'
```

**現在サポートされている AZ ID を調べるには:** AgentCore の VPC 設定ガイド https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-vpc.html の "Supported Availability Zones" セクションを参照してください。この表には、リージョンごとの AZ ID（例: `use1-az1`、`usw2-az2`）が載っています。AZ 名と AZ ID の対応はアカウントによって異なるため、AZ 名ではなく AZ ID を使用してください。

サブネットがサポートされていない AZ にある場合、デプロイは失敗します。サポートされている AZ のサブネットを使用してください。

**ベストプラクティス:** 可用性を高めるために、サポートされている異なる AZ に、少なくとも 2 つのサブネットを使用してください。

---

## ステップ 2: セキュリティグループを設定する

セキュリティグループは、エージェントが接続できる先を制御します。接続先に応じて設定してください。

### RDS PostgreSQL に接続する場合

**AgentCore エージェントのセキュリティグループ**（アウトバウンドルール）:

```
Type: Custom TCP
Port: 5432
Destination: RDS のセキュリティグループ ID（CIDR ではない）
```

**RDS のセキュリティグループ**（インバウンドルール）:

```
Type: PostgreSQL
Port: 5432
Source: AgentCore エージェントのセキュリティグループ ID
```

```bash
# エージェント用のセキュリティグループを作成する
aws ec2 create-security-group \
  --group-name agentcore-agent-sg \
  --description "AgentCore agent security group" \
  --vpc-id vpc-12345678

# RDS に到達するためのアウトバウンドルールを追加する
aws ec2 authorize-security-group-egress \
  --group-id sg-agent123 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-rds456

# RDS のセキュリティグループにインバウンドルールを追加する
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds456 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-agent123
```

### 内部 API（HTTP/HTTPS）に接続する場合

**AgentCore エージェントのセキュリティグループ**（アウトバウンドルール）:

```
Type: HTTPS, Port: 443, Destination: API のセキュリティグループまたは CIDR
Type: HTTP, Port: 80, Destination: API のセキュリティグループまたは CIDR（必要な場合）
```

---

## ステップ 3: エージェントを VPC 用に設定する

### 新規プロジェクト

```bash
agentcore create \
  --name MyAgent \
  --defaults \
  --network-mode VPC \
  --subnets subnet-abc123,subnet-def456 \
  --security-groups sg-agent123
```

### 既存プロジェクト

```bash
agentcore add agent \
  --name MyAgent \
  --network-mode VPC \
  --subnets subnet-abc123,subnet-def456 \
  --security-groups sg-agent123
```

または、`agentcore/agentcore.json` を直接編集して、ランタイムのエントリに `networkMode` と `networkConfig` フィールドを追加します。

```json
{
  "runtimes": [
    {
      "name": "MyAgent",
      "networkMode": "VPC",
      "networkConfig": {
        "subnets": ["subnet-abc123", "subnet-def456"],
        "securityGroups": ["sg-agent123"]
      }
    }
  ]
}
```

`agentcore.json` の先頭にある `$schema` の URL（`https://schema.agentcore.aws.dev/v1/agentcore.json`）により、サブネットやセキュリティグループの ID のパターンを含め、すべてのフィールドで IDE のオートコンプリートと検証が使えます。

### デプロイ

```bash
agentcore deploy -y
```

---

## VPC からのインターネットアクセス

> [!WARNING]
> AgentCore を VPC に接続しても、デフォルトではインターネットアクセスは提供されません。
> パブリックサブネットでは、AgentCore の ENI にインターネットアクセスは提供されません。
> VPC モードからインターネットに到達するには、NAT ゲートウェイを備えたプライベートサブネットを
> 使用する必要があります。

**インターネットと VPC の両方にアクセスするためのアーキテクチャ:**

```
AgentCore エージェント（プライベートサブネット）
    ↓ アウトバウンドトラフィック
NAT ゲートウェイ（パブリックサブネット）
    ↓
インターネットゲートウェイ
    ↓
インターネット
```

```bash
# パブリックサブネットに NAT ゲートウェイを作成する
aws ec2 create-nat-gateway \
  --subnet-id subnet-public123 \
  --allocation-id eipalloc-12345678

# プライベートサブネットのルートテーブルを更新して NAT ゲートウェイを使うようにする
aws ec2 create-route \
  --route-table-id rtb-private123 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-12345678
```

---

## 完全にプライベートな VPC（インターネットなし）

VPC にインターネットアクセスがない場合は、AWS サービス用の VPC エンドポイントが必要です。これらのエンドポイントは、インターネットアクセスがない場合は**必須**であり、NAT ゲートウェイがある場合でも、NAT ゲートウェイのデータ処理料金を避けるために**強く推奨**されます。

```bash
# ECR Docker エンドポイント（コンテナイメージの取得に必須）
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.REGION.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc123 \
  --security-group-ids sg-agent123

# ECR API エンドポイント（コンテナイメージの取得に必須）
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.REGION.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc123 \
  --security-group-ids sg-agent123

# S3 ゲートウェイエンドポイント（必須 — ECR はイメージレイヤーを S3 に保存する）
# これは無料のゲートウェイエンドポイントです。これがないと、ECR のイメージ更新が
# NAT 経由でルーティングされ、データ処理料金が発生します。
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.REGION.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-private123

# CloudWatch Logs（エージェントのログ出力に必須）
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.REGION.logs \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc123 \
  --security-group-ids sg-agent123
```

---

## コールドスタート時の接続性チェックリスト

よくあるパターン: `UpdateAgentRuntime` が READY を返し、ネットワーク設定も正しく見えるのに、呼び出しが 502 を返したりハングしたりします。リクエストはコンテナに届きません。これは、ほぼ間違いなく、新しい VM は起動できるものの、トラフィックを受け付ける準備を完了するために必要な処理を完了できない状態を意味します。

コールドスタートする VM は、次の AWS サービスのエンドポイントへのアウトバウンド HTTPS（ポート 443）を必要とします。パブリックまたは NAT 経由の VPC では、正しく設定された NAT ゲートウェイがこれらすべてをカバーします。完全にプライベートな VPC では、次のそれぞれに、インターフェイス VPC エンドポイントまたはゲートウェイエンドポイントが必要です。

- `com.amazonaws.<region>.ecr.api` — イメージのメタデータを取得する
- `com.amazonaws.<region>.ecr.dkr` — コンテナレイヤーを取得する
- `com.amazonaws.<region>.s3`（ゲートウェイエンドポイント）— ECR のレイヤーは S3 にある
- `com.amazonaws.<region>.logs` — CloudWatch Logs を出力する
- `com.amazonaws.<region>.monitoring` — CloudWatch メトリクスを出力する
- `com.amazonaws.<region>.sts` — 実行ロールを引き受ける

これに加えて、エージェントのツールや依存関係が必要とするエンドポイント（Bedrock、DynamoDB、Secrets Manager など）も必要です。

### セキュリティグループのアウトバウンドルール

エージェントのセキュリティグループには、各 VPC エンドポイントのプレフィックスリストの 443 に到達するためのアウトバウンドルール、またはエンドポイントに直接到達できる場合は `0.0.0.0/0` が必要です。

```bash
aws ec2 authorize-security-group-egress \
  --group-id sg-agent123 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

エグレスをより厳しく絞る場合（特定のエンドポイントのプレフィックスリストや CIDR ブロックに限定する場合）は、上記のすべてのエンドポイントがカバーされていることを再度確認してください。

### NACL — 落とし穴

ネットワーク ACL は**ステートレス**です。アウトバウンドの 443 を許可するセキュリティグループは、暗黙的に応答トラフィックも許可します。NACL は許可しません。

サブネットが制限の厳しい NACL を使っている場合は、両方向を明示的に許可する必要があります。

- **アウトバウンド:** 宛先への TCP 443 を許可する
- **インバウンド:** 宛先からの**エフェメラルポート 1024〜65535**（TCP）を許可する — これらは戻りトラフィックのポートです

インバウンドのエフェメラルポートのルールを忘れると、「接続できるときもあれば、ハングするときもある」という症状がそのまま現れます。TCP のハンドシェイクは成功しますが（SYN が出ていき、SYN-ACK は小さい番号のポート範囲で戻ってきます）、エフェメラルポート上の実際のデータの応答が破棄されるためです。

### Transit Gateway とカスタムのエグレス

サブネットがアウトバウンドを Transit Gateway 経由で、中央のファイアウォール、NAT、ネットワーク仮想化レイヤーにルーティングしている場合は、TGW アタッチメントとその先に、インターネット（または各 VPC エンドポイント個別）への有効なルートが必要です。

TGW のルートが欠けているときの症状:

- 呼び出しが、クライアント側のタイムアウト（デフォルトの Lambda クライアントでは約 300 秒）いっぱいまでハングする
- 502 も `ConnectionClosedError` も出ない — リクエストがただ戻ってこない
- 同じサブネット/セキュリティグループにあるテスト用 EC2 からの `ping` は成功するのに、実際の呼び出しは成功しない
- ウォーム状態の環境（すでに初期化済みで、必要なエグレスもすべて完了している）は成功し、新しいコールドスタートは失敗する

ここでは、EC2 からテストするパターンが有効です。同じサブネットで同じセキュリティグループを持つ t3.micro を起動し、`curl https://s3.<region>.amazonaws.com`、`curl https://ecr.<region>.amazonaws.com` などを試します。これらのどれかがハングまたは失敗する場合、エージェントもコールドスタートに失敗します。

### VPC モードではコールドスタート時間が長くなると想定する

VPC モードでは、コンテナイメージの取得とアプリケーションの起動に加えて、ENI のアタッチとセットアップの時間がコールドスタートに加わります。設定したばかりの VPC での最初の呼び出しは、パブリックモードよりも明らかに遅くなります。

対策は、あらゆるコールドスタートのレイテンシーに対するものと同じです。セッションを再利用し、イメージを小さく保ち、重い初期化は後回しにします。`agents-harden` の Initialization time セクションを参照してください。

---

## トラブルシューティング

**RDS や内部 API への接続がタイムアウトする:**

1. セキュリティグループのルールを確認する — エージェントの SG からのアウトバウンドと、接続先の SG へのインバウンド
2. ルートテーブルを確認する — プライベートサブネットは、NAT ゲートウェイ（インターネット向け）にルーティングされているか、接続先への直接ルートを持っている必要がある
3. VPC で DNS 解決が有効になっていることを確認する: `aws ec2 describe-vpc-attribute --vpc-id vpc-12345678 --attribute enableDnsSupport`

**デプロイ中の "Unsupported Availability Zone" エラー:**
サブネットが、AgentCore がサポートしていない AZ にあります。AZ 名ではなく AZ ID を確認し、サポートされている AZ のサブネットを使用してください。

**VPC を設定したあと、エージェントがインターネットに到達できない:**
パブリックサブネットを使っているか、NAT ゲートウェイがありません。パブリックサブネット内の AgentCore の ENI は、インターネットアクセスを得られません。NAT ゲートウェイを備えたプライベートサブネットを使用してください。

**VPC エンドポイントを使うときの "AccessDenied":**
実行ロールに、VPC エンドポイントの先にあるサービスに対する権限がありません。エンドポイントのリソースポリシーと、実行ロールの IAM ポリシーを確認してください。

**Code Interpreter がパブリックエンドポイントの呼び出しでタイムアウトする:**
エージェントが VPC 内にある場合は、Code Interpreter にも VPC の設定が必要です。同じサブネットと、インターネットアクセス用の NAT ゲートウェイで設定してください。

**DNS 解決の失敗:**
VPC で DNS 解決と DNS ホスト名を有効にします。

```bash
aws ec2 modify-vpc-attribute --vpc-id vpc-12345678 --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id vpc-12345678 --enable-dns-hostnames
```

## 出力

- サブネットの AZ の検証結果
- 特定の接続先（RDS、内部 API など）に対するセキュリティグループのルール
- VPC モードを設定するための CLI コマンド
- インターネットアクセスが必要な場合の NAT ゲートウェイのセットアップ
- 完全にプライベートなデプロイのための VPC エンドポイントの一覧

## 品質基準

- サブネットの AZ ID が、サポートされている AZ に対して検証されている（AZ 名ではない — 名前はアカウントによって異なる）
- セキュリティグループのルールが両方向をカバーしている（エージェントのアウトバウンド + 接続先のインバウンド）
- インターネットアクセスには NAT ゲートウェイを推奨している（パブリックサブネットではない — AgentCore の ENI はパブリック IP を持たない）
- 完全にプライベートなデプロイのための VPC エンドポイントの一覧が漏れなくそろっている
- 開発者が、`networkMode: VPC` が主にアウトバウンドトラフィックに影響することを理解している
