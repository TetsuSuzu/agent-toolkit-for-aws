---
name: agents-get-started
description: >
  開発者が新しいエージェントプロジェクトを作成したいとき、または AgentCore を
  使い始めたいときに使います。フレームワークの選択、プロジェクトのスキャフォールディング、
  最初のデプロイ、最初の呼び出しを扱います。トリガー: "build an agent"、"create an
  agent"、"get started"、"new project"、"agentcore create"、"which
  framework"、"Strands vs LangGraph"、"hello world agent"、"first agent"、
  "create MCP server"、"host MCP server"、"agentcore dev"、"dev server"、
  "what port"、"local development"。
  既存のプロジェクトへの機能追加には使いません — agents-build または
  agents-connect を使います。移行の文脈での Strands と LangGraph の比較は、
  ここではなく agents-build に振り分けます。既存の MCP サーバーへの接続は、
  ここではなく agents-connect に振り分けます。
allowed-tools: Read Grep Glob Bash
metadata:
  type: skill
  version: "1.0.0"
  author: aws-agentcore
  requires-cli: ">=0.9.0"
---

> **注意:** これは `SKILL.md` の日本語参考訳です。スキルとして読み込まれるのは英語版の `SKILL.md` だけです。リンク先は、日本語版の `references-ja/` を指しています。

# get-started

開発者を、ゼロの状態から、AWS 上で動作するエージェントまで案内します。

## 使うタイミング

- 開発者が AWS 上でエージェントを構築したいが、どこから始めればよいか分からない
- 開発者が新しい AgentCore プロジェクトを作成したい
- 開発者がフレームワーク（Strands、LangGraph、GoogleADK、OpenAI Agents）を選んでいる
- 開発者が `agentcore create` を実行したばかりで、次に何をすべきか知りたい

次の場合は使わないでください:

- 環境/前提条件の問題（CLI が見つからない、認証情報が壊れている）→ `agents-debug` を使います
- 既存のプロジェクトへの機能追加（Memory、ツール、ポリシー）→ `agents-build` または `agents-connect` を使います
- 既存の Bedrock Agent の移行 → `agents-build` を使います（[`references-ja/migrate.md`](../agents-build/references-ja/migrate.md) を読み込む）

## 入力

`$ARGUMENTS` には次のものを指定できます。

- フレームワークの希望: "using LangGraph"、"with Strands"
- プロトコル: "MCP server"、"A2A"
- エージェントにさせたいことの説明: "a customer support agent"
- 空 — スキルがフレームワークの選択を案内します

## 手順

### ステップ 0: CLI のバージョンを確認する

```bash
agentcore --version
```

このスキルには v0.9.0 以降が必要です。

バージョンが古い場合:
> AgentCore CLI が古くなっています（vX.Y.Z が見つかりました。v0.9.0 以上が必要です）。

更新の実行を提案します: `agentcore update`。更新が完了したら、続行する前に、バージョンが 0.9.0 以上であることをもう一度確認してください。開発者がすでに伝えた内容（フレームワークの希望、プロジェクト名、作りたいもの）は保持して、繰り返させないようにします。

`agentcore` が見つからない場合:
> AgentCore CLI がインストールされていません。`npm install -g @aws/agentcore` を実行してください（Node.js 20 以上が必要です）。
> インストールで困っている場合は、`agents-debug` スキル（[`references-ja/doctor.md`](../agents-debug/references-ja/doctor.md) を読み込む）を実行して、環境を診断できます。

### ステップ 1: 意図を判断する — 検討中か、作成する準備ができているか

フレームワークの選択に入る前に、開発者がどの段階にいるのかを把握します。

**開発者に尋ねます:**「選択肢を検討中ですか(フレームワークを比較したり、AgentCore が何をするのかを理解したりしている段階)、それともプロジェクトを作成する準備ができていますか?」

- **検討中** → ステップ 2（フレームワークの比較）に進みます。選択肢を示し、質問に答えて、待ちます。準備ができたと伝えられるまで、`create` コマンドを組み立てないでください。
- **作成する準備ができている** → ステップ 3（プロジェクトの作成）に進みます。すでにフレームワークを指定している場合は、ステップ 2 を完全にスキップします。
- **すでにプロジェクトがある** → カレントディレクトリで `agentcore/agentcore.json` を探します。見つかったら、それを読み、ステップ 5（次にやること）に進みます。スキャフォールドをやり直さないでください。

開発者の意図が `$ARGUMENTS` から明らかな場合（例: "create a Strands agent called MyBot"）は、すぐにステップ 3 に進みます。

### ステップ 2: フレームワークの選択

**まず、会話のコンテキストを確認します。** 開発者がこの会話の中ですでにフレームワークについて話し合っている場合（たとえば、以前のスキルの呼び出しで）は、表を再び全部示さないでください。話し合った内容を要約して、決まったか、何か変わったかを尋ねます。

フレームワークについて話すのが初めての場合は、選択肢を示します。

**サポートされているフレームワーク（CLI がスキャフォールドするもの、Python）:**

| フレームワーク | CLI の値 | 適している用途 |
|---|---|---|
| Strands | `Strands` | AWS ネイティブ、最も簡単な方法、AgentCore との統合が最も優れている |
| LangGraph | `LangChain_LangGraph` | 複雑なグラフベースのワークフロー、LangChain への既存の投資がある場合 |
| Google ADK | `GoogleADK` | すでに Google のエージェントツールキットを使っているチーム |
| OpenAI Agents | `OpenAIAgents` | すでに OpenAI のエージェント SDK を使っているチーム |

**開発者に選んでもらいます。** 選択肢を示して、選択を待ちます。特に希望がないと明示されない限り、デフォルトを仮定しないでください。

> **名前に関する注意:** CLI のフラグの値は、`--framework` に渡す正確な文字列です。文章では、短い名前を使います。

**デフォルトの推奨**（開発者が「希望はない」「あなたが選んで」と言った場合のみ）: Strands — AgentCore との統合が最も緊密で、サンプルとドキュメントが最も多い、AWS ネイティブのフレームワークです。

**確認すべき主な判断ポイント:**

- 「LangGraph や OpenAI Agents で書いた既存のエージェントのコードはありますか?」→ そのフレームワークを使う
- 「条件分岐を伴う複雑なグラフベースのワークフローが必要ですか?」→ LangGraph
- 「希望なく、新規に始めますか?」→ Strands

#### 一覧にないフレームワークは?

開発者が、上の表にないフレームワークについて尋ねた場合は、次のように対応します。

| 質問されたもの | 伝える内容 |
|---|---|
| **CrewAI、AutoGen、Semantic Kernel** | CLI ではスキャフォールドされませんが、BYO Container のパス(下記)で使えます。AgentCore Runtime はフレームワークに依存せず、HTTP の契約を実装するコードなら何でも動作します。 |
| **Anthropic SDK / Claude Agent SDK** | これはモデルの SDK で、エージェントのフレームワークではありません。どのフレームワーク（Strands、LangGraph など）の中でも、単体でも使えます。単体で使う場合は、Runtime の契約を実装したコンテナでラップします。 |
| **Claude Code / Cursor / Copilot** | これらは IDE のツールで、エージェントのフレームワークではありません。エージェントのコードを*書く*場所であって、デプロイするものではありません。エージェント自体には、上の表からフレームワークを選んでください。 |
| **LangChain(LangGraph なし)** | LangChain はライブラリで、LangGraph はその上に構築されたエージェントのフレームワークです。CLI がスキャフォールドするのは LangGraph です。素の LangChain のチェーンを使っている場合は、BYO Container のパスが使えます。 |
| **カスタム / 自前のフレームワーク** | BYO Container のパス — 下記を参照。 |

**BYO Container のパス(任意のフレームワーク、任意の言語):**

CLI がスキャフォールドしないフレームワークや言語でも、AgentCore Runtime は、HTTP の契約（`POST /invocations`、`GET /ping`）を実装したコンテナなら何でも受け付けます。ワークフロー:

1. `agentcore create --name <ProjectName> --defaults` で、プロジェクトの構造をスキャフォールドする
2. `agentcore add agent --type byo --build Container --language <Language> --code-location <path>` で、自分のコードを登録する
3. エージェントをビルドして実行する `Dockerfile` を書く
4. `agentcore deploy` が、ECR へのプッシュ、CDK のインフラ、ランタイムの作成を処理する

**言語ごとの注意:**

| 言語 | 推奨される方法 |
|---|---|
| Java (Spring Boot) | [Spring AI SDK for AgentCore](https://aws.amazon.com/blogs/machine-learning/spring-ai-sdk-for-amazon-bedrock-agentcore-is-now-generally-available) — Runtime の契約、SSE ストリーミング、ヘルスチェックを処理します。`--language Other --build Container` を使います。 |
| JavaScript / TypeScript | Express/Fastify などで Runtime の契約を実装します。`--language TypeScript --build Container` を使います。 |
| Go、Rust、.NET、その他 | Runtime の HTTP の契約を実装します。`--language Other --build Container` を使います。 |

このスキルの残りの部分（デプロイ、ステータス、ログ、呼び出し）は、コンテナが正しくビルドできてから適用されます。

#### フレームワークとモデルプロバイダー — よくある混同

フレームワークは、エージェントがどのようにオーケストレーションを行うか(Strands、LangGraph など)です。モデルプロバイダーは、どの LLM を呼び出すか(Bedrock、Anthropic、OpenAI、Gemini)です。これらは独立した選択です。

- Strands + Bedrock（デフォルト）— すべて AWS ネイティブ
- Strands + Anthropic — オーケストレーションは Strands、モデルは Anthropic の API を直接使う
- LangGraph + Bedrock — オーケストレーションは LangGraph、モデルは Bedrock
- OpenAI Agents + OpenAI — すべて OpenAI

開発者が「Claude を使いたい」と言った場合は、フレームワークではなく、モデルプロバイダー(Bedrock または Anthropic)のことです。「LangGraph を使いたい」と言った場合は、フレームワークのことです。

### ステップ 3: プロジェクトを作成する

開発者の選択に基づいて、`agentcore create` コマンドを組み立てます。

**コマンドを組み立てる前に — プロジェクト名を検証します。** CLI は遅れて失敗します。名前が無効な場合、プロンプトを一通り進めたり、完全なコマンドを組み立てたりした*後*にエラーが表示されます。往復を省くために、次のルールを事前に確認してください。いずれかのルールに反する場合は、その名前を却下して、新しい名前を尋ねます。

- **23 文字以内**(多くの開発者が想定するより短いです。`MyCustomerSupportAgent` は 22 文字で収まり、`CustomerSupportChatbot` は 22 文字で収まり、`MyCustomerSupportBotApp` は 23 文字でぎりぎり収まり、`MyCustomerSupportChatBot` は 24 文字で**失敗します**)
- **英数字のみ** — ハイフン、アンダースコア、ドット、スペースは使えません
- **英字で始まる**こと

上限に近い場合は、文字数を声に出して伝えてください。「その名前は 24 文字です。CLI はプロジェクト名を 23 文字までに制限しています。`<suggestion>` に短縮しますか?」CLI のエラーメッセージが分かりやすいだろうと期待して、無効な名前のままコマンドを実行しないでください。常に分かりやすいとは限らず、開発者の頭の中のモデルが、以降のコマンドで誤ったものになります。

**コマンドを組み立てたら、開発者が実行する前に、確認のために提示します。** すべてのフラグを含む完全なコマンドを示し、各選択が何を意味するかを説明します。開発者が確認または調整するまで、待ってください。

提示の例:

> これまでのお話に基づいて、次のコマンドをお勧めします。
>
> ```bash
> agentcore create --name MyAgent --framework Strands --model-provider Bedrock --build CodeZip --memory none
> ```
>
> これは、Bedrock のモデルを使う Strands のエージェントを作成し、コードの zip としてデプロイします(Docker は不要です)。Memory は後から追加できます。
>
> これを実行しますか、それとも何か変更しますか?

コマンドを自動で実行しないでください。提示して、待ちます。

**最小構成(デフォルト — Strands、Bedrock、CodeZip、Memory なし):**

```bash
agentcore create --name <ProjectName> --defaults
```

**オプションを指定する場合:**

```bash
agentcore create \
  --name <ProjectName> \
  --framework <Framework> \
  --model-provider Bedrock \
  --build CodeZip \
  --memory none
```

**フラグのリファレンス:**

| フラグ | 値 | デフォルト |
|---|---|---|
| `--name` | 英数字、最大 23 文字 | 入力を求められる |
| `--framework` | `Strands`、`LangChain_LangGraph`、`GoogleADK`、`OpenAIAgents` | 入力を求められる |
| `--protocol` | `HTTP`、`MCP`、`A2A` | `HTTP` |
| `--build` | `CodeZip`、`Container` | `CodeZip` |
| `--model-provider` | `Bedrock`、`Anthropic`、`OpenAI`、`Gemini` | 入力を求められる |
| `--memory` | `none`、`shortTerm`、`longAndShortTerm` | 入力を求められる |
| `--network-mode` | `PUBLIC`、`VPC` | `PUBLIC` |
| `--dry-run` | — | 作成せずにプレビューする |

**選択のガイダンス:**

- **プロトコル:** 開発者が、MCP のツール提供や A2A のエージェント間通信を特に必要とする場合を除き、`HTTP` を使います
- **ビルド:** 開発者が、独自のシステム依存関係を必要とする場合を除き、`CodeZip` を使います(CodeZip のほうがデプロイが速く、ローカルに Docker が不要です)
- **モデルプロバイダー:** 別のプロバイダーを使う特別な理由がない限り、`Bedrock` を使います(Bedrock は API キーの管理が不要です)
- **Memory:** まず `none` から始めます — 必要になったときに、`agents-build`([`references-ja/memory.md`](../agents-build/references-ja/memory.md) を読み込む)で後から追加できます

### ステップ 4: 作成されたものを説明する

プロジェクトができたら、`agentcore/agentcore.json` と生成されたコードを読み、プロジェクトの構造を説明します。

以下のレイアウトは CLI v0.9.x のものです。CLI のバージョンが異なる場合は、`tree <ProjectName>/ -L 3` を実行して、実際に生成された構造を確認し、それに基づいて説明します。

```
<ProjectName>/
├── agentcore/
│   ├── agentcore.json      ← プロジェクトの設定（エージェント、リソース）
│   ├── aws-targets.json    ← AWS アカウント + リージョン
│   ├── .env.local          ← ローカルの環境変数（gitignore 対象）
│   └── cdk/                ← CDK のインフラ（自動管理。編集しない）
└── app/
    └── <AgentName>/
        ├── main.py          ← エージェントのコード — ここで構築する
        ├── mcp_client/      ← 組み込み済みのサンプル MCP クライアント（下記の注意を参照）
        └── pyproject.toml   ← Python の依存関係
```

**強調すべき主なファイル:**

- `app/<AgentName>/main.py` — エージェントのエントリーポイント。開発者がツール、システムプロンプト、ロジックを追加する場所です。
- `agentcore/agentcore.json` — プロジェクトの設定。リソースは、`agentcore add` コマンドでここに追加されます。
- `agentcore/.env.local` — ローカルの環境変数。デプロイ後、ローカル開発用に、リソース ID がここに書き込まれます。

**スキャフォールドされた MCP クライアントについての注意。** `main.py` は `mcp_client/client.py` から `get_streamable_http_mcp_client()` をインポートし、それを `tools` に追加します。新しいプロジェクトでは、このクライアントは公開されているサンプルの MCP エンドポイントを指しているため、`agentcore dev` がすぐに動作します。注意すべき点が 2 つあります。

1. **まだデプロイされていない Gateway に向け直すと、通知なしに何も起きなくなります。** よくあるのは、サンプルのエンドポイントを `os.getenv("AGENTCORE_GATEWAY_<NAME>_URL")` に置き換えるケースです。この環境変数は、`agentcore deploy` の後にしか設定されません。開発者が向け直して、デプロイの前に `agentcore dev` を実行すると、`get_streamable_http_mcp_client()` は URL が `None` のクライアントを返し、エージェントは MCP ツールが 0 個の状態で起動します。エラーも警告も出ません。ガードのパターンは、`agents-connect` の「ローカル開発での制約」セクションを参照してください: `if not GATEWAY_URL: tools = []`。
2. **開発者が MCP ツールをまったく必要としない場合は、** `mcp_clients` のリストと、それを `tools` に追加するループを削除します。スキャフォールドに含まれているのは、利便性のためで、必須ではありません。

デプロイ後の、Gateway を使った MCP クライアントの正しいパターンは、`agents-connect`（Path A）の参考クライアントコードに示されています。

### ステップ 5: ローカル開発

```bash
agentcore dev
```

これはローカルの開発サーバーを起動します。開発者は、すぐにエージェントと対話できます。

**開発サーバーがバインドするポート**(`curl` の呼び出しをスクリプトにする場合や、別のプロセスからテストする場合に重要です):

| プロトコル | デフォルトのポート |
|---|---|
| HTTP | `8080` |
| MCP | `8000` |
| A2A | `9000` |

CLI は起動時に、バインドしたポートと URL を表示します。ハードコードせずに、常に CLI の出力から実際の値を読み取ってください。**デフォルトのポートがすでに使用中の場合**、CLI は自動的にインクリメントします（例: 8080 → 8081 → 8082）。そのため、2 つ目の開発セッションや、前回の実行から残っているプロセスによって、通知なしにポートがずれることがあります。`agentcore dev --port <N>` で固定するか、呼び出しが connection-refused や exit-code-7 のエラーで失敗し始めたら、`ps` を grep するか、CLI のバナーを確認してください。

**伝えておくべき重要な制限:**

- Memory は `agentcore dev` では利用できません — デプロイが必要です
- Gateway の URL はローカルでは利用できません — デプロイが必要です
- ローカルサーバーは、プロジェクトに設定されたモデルプロバイダーを使います

### ステップ 6: 最初のデプロイ

開発者の準備ができたら:

```bash
agentcore deploy
```

これは次のことを行います。

1. 作成される AWS リソースのプレビューを表示する
2. 確認を求める
3. CDK でビルドしてデプロイする

**最初のデプロイには 3〜5 分かかります。** 2 回目以降は速くなります。

デプロイ後に、呼び出し方を示します。

```bash
agentcore invoke "Hello, what can you do?"
```

そして、状態の確認方法を示します。

```bash
agentcore status
```

### ステップ 7: 次のステップ

開発者が作りたいと言っているものに基づいて、論理的に次のスキルを提案します。

| 開発者の意図 | 次のスキル | コマンドのヒント |
|---|---|---|
| 「アプリからどうやって呼び出しますか?」 | `agents-build` | `agentcore fetch access` |
| 「物事を覚えさせたい」 | `agents-build` | `agentcore add memory` |
| 「外部の API を呼び出させたい」 | `agents-connect` | `agentcore add gateway` |
| 「できることを制限したい」 | `agents-connect` | `agentcore add policy-engine` |
| 「品質を測定したい」 | `agents-optimize` | `agentcore add evaluator` |
| 「本番に移行したい」 | `agents-harden` | 本番準備のチェックリスト |
| 「複数のエージェントを連携させたい」 | `agents-build` | `agentcore create --protocol A2A` |
| 「VPC に置く必要がある」 | `agents-build` | `agentcore create --network-mode VPC` |

圧倒しないようにしてください。開発者が実際に尋ねたことに基づいて、1 つか 2 つの次のステップを提案します。

### 例のウォークスルー

タスクの形で書かれたプロンプト(例: 「カスタマーサポートのエージェントを作りたい」)には、対応する例のリファレンスを読み込みます。

| 開発者のタスク | リファレンス |
|---|---|
| カスタマーサポート、チャットボット、ポリシーに関する質問への回答 | [`references-ja/example-support-agent.md`](references-ja/example-support-agent.md) |

よくあるパターンが出てきたら、このスキルのリファレンスのディレクトリに例を追加できます。

## 出力

- 「エージェントを作りたい」から、デプロイされて動作するエージェントまでの明確な道筋
- 開発者の選択に合わせた `agentcore create` コマンド
- 生成されたプロジェクトの構造の説明
- 開発者の意図に基づく、具体的な次のステップ

## 品質基準

- `agentcore create` コマンドが、CLI v0.9.1 の有効なフラグのみを使っている
- フレームワークの推奨が、一般的なデフォルトではなく、開発者のコンテキストに基づいている
- 開発者が、生成された各ファイルが何をするのかを理解している
- 次のステップが、すべての機能の一般的な一覧ではなく、開発者が作りたいものに即している
