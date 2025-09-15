---
title: "Alibaba cloud と Qwen LLM で LINE Chatbot を作成する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["alibaba", "alibabacloud", "terraform", "terragrunt", "qwen"]
published: false
---

## はじめに

こんにちは、今回は AWS, Azure, GCP の次に名前が上がるにも関わらず、業務上では殆どお目にかかれないクラウド、Alibaba Cloud で遊びたくなったので LINE 上で Qwen を使ったチャットボットを作成してみました。

コンプライアンス等の要請から会社の業務用ツールとは完全に別個にしたいのと、友人間の流れみたいなところもあったのでプライベートでアカウントを取得して LINE でボットを作ってみることにしました。

地政学的な問題から避けられがちな Alibaba Cloud ですが、実は世界 25+ リージョン・90 以上のアベイラビリティゾーンを展開しており、中国・ASEAN とのビジネスを行う上では選択肢の第一候補として挙がってくるものだと認識しています。
喫緊で必要というわけではないですが、触れておいて損はなさそうですね。

筆者は普段 AWS を利用しているため、所々で AWS との比較が出てきますがご容赦くださいませ。

## LINE Messaging API を用意する

LINE bot は LINE Messaging API を利用して設定します。
Messaging API により、Bot 宛てに送られたメッセージやイベントを Webhook 経由で受信し、返信やプッシュ送信することが可能です。
詳しくは以下の記事を参考にしてください。チャネルシークレットとチャネルアクセストークン両方利用するので控えておきます。

https://zenn.dev/kou_pg_0131/articles/line-push-text-message

公式ドキュメントはこちらです。必要に応じて参照してください。

https://developers.line.biz/ja/docs/messaging-api/building-bot/

Bot 名は「わんわんお」にしました。

## Alibaba Cloud のアカウントを用意する

[Alibaba Cloud](https://www.alibabacloud.com/ja?_p_lc=1) にアクセスし、右上の「ログイン」からサインアップします。住所やクレカ情報を入力してアカウントを作成します。

Alibaba Cloud に登録すると各種リソースの無料枠が付与されます。
今回は積極的に利用しないですが、今後インスタンスを立てたり検証する際に使ってみようと思います。

また、Alibaba Cloud では一部サービスは初回に有効化手続きが必要です。例えば Terraform からリソースを作成する際、有効化していないと以下のようなエラーが出ることがあります。

```
SDKError:
   StatusCode: 401
   Code: InvalidAccessKeyId
   Message: Your SLS service has not been opened.
```

今回利用するサービスで有効化が必要なのは Simple Log Service (SLS) と Object Storage Service (OSS) です。

コンソール (Management Console) から OSS を開くと、有効化要求画面が現れます。「今すぐ有効にする」をクリックして画面の指示に従い、有効化手続きを行います。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-info-activation.png)

SLS に関しても同様ですが、途中で以下のような謎のモーダルが出現することがあります。

![](/images/create-line-chatbot-with-alicloud-and-qwen/sls-purchase-billing-url.png)

URL をブラウザのアドレスバーにコピペしてアクセスすると、クレジットカードの検証画面に移るので、引き続き画面の指示に従っていくと有効化手続きが完了します。

一応ですが、クラウドを個人で運用する際は **Budget 設定** 行うの忘れずに。マイニング専用アカウントになってからでは遅いので。。。

## Qwen の有効化

Qwen は Alibaba Group が開発した大規模言語モデルで、Alibaba Cloud の [**Model Studio**](https://modelstudio.console.alibabacloud.com/) から利用できます。

Model Studio にアクセスすると、Qwen や他のモデルを管理するダッシュボードが表示されます。右上に Mainland China Edition / International Edition と書かれたドロップダウンがありますが、エンドポイントの所在地が中国大陸内外で分かれているようです。ひとまず今回は International Edition を利用します。

Model Studio は無償試用制度があり、一定の条件下で各種モデルを無料で呼び出すことが可能です。ログイン後右上にある `New User Offer ... Activate Now` をクリックして利用を開始します。詳しくは[公式ドキュメント](https://modelstudio.console.alibabacloud.com/?tab=doc#/doc/?type=model&url=2766612)を参照してください。

次に右上の `Get API Key` から Secret Key を取得します。クリックすると Key の一覧ページに遷移するので `Create API Key` から Default workspace に鍵を一つ追加してみました。以下のようにエントリが追加されたので、API Key をコピーして控えます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/qwen-dashboard-apikey.png)

そのまま右の Workspaces をクリックしてワークスペースの一覧ページに遷移し Authorization & Throttling Settings を確認。

![](/images/create-line-chatbot-with-alicloud-and-qwen/qwen-dashboard-workspaces.png)

クリックすると利用可能なモデルの一覧が現れるので、その中から使いたいモデルの名前を確認しておきます。

## Terragrunt 用の backend を用意する

Terragrunt で Alibaba Cloud のリソースを操作するためのユーザと tfstate 用の OSS bucket の整備をコンソール上で行います。

ユーザの作成と権限付与は RAM (Resource Access Management) で行います。RAM はユーザ等のエンティティと権限を管理する機能であり、AWS IAM に相当します。

まず、コンソールから RAM を開いてユーザを作成します。ユーザの作成ボタンを押すと以下のような画面が表示されます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-create.png)

Using permanent AccessKey to access を ON にして他のオプションをデフォルトにします。ログイン名と表示名を記載して OK ボタンを押すと、以下のような画面が表示されます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-create-credentials.png)

CSV File をダウンロードして控えておきましょう。ここに書いてある AccessKey ID と AccessKey Secret を認証情報として terraform を実行します。

ユーザを作成したら、次は権限付与を行います。権限管理タブを開き、Grant Permission ボタンをクリック。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant.png)

モーダルが開きます。今回は RAM 関連の操作も行うので、AdministratorAccess のマネージドポリシーを選択し付与します。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant-detail.png)

AdministratorAccess のトグルを ON にしようとすると警告が出ますが、今回は検証目的なので見逃します。実務上で紐づけるなら VPC 内の ECS インスタンスに開発環境作成して RAM ロールをアタッチするのがベストですかね。Admin 権限でアクセスキーを外に出すのは本番環境では正直避けたいところです。

次に tfstate 保存先の OSS bucket を作成します。
コンソールから OSS を開き、バケット一覧画面を表示しましょう。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-create-bucket.png)

バケットの作成をクリックします。細かい設定は置いといて、公開アクセス禁止の状態で東京リージョンに作成します。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-create-bucket-detail-3.png)

コンソールでの準備が完了したので次は Terragrunt の設定です。以下のような構成でファイルツリーを構成します。

```
infra/
├── live/dev/bot/
│   └── terragrunt.hcl
├── modules/bot/
│   └── .gitkeep
└── root.hcl
services/
└── ...
```

`root.hcl` の中に provider および backend 設定を記述します。ここで先ほど設定した OSS bucket と RAM ユーザを利用します。

```hcl
remote_state {
  backend = "oss"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }

  config = {
    bucket  = "linebot-tf-backend"
    key     = "${path_relative_to_include()}/terraform.tfstate"
    acl     = "private"
    region  = "ap-northeast-1"
    encrypt = true

    access_key = get_env("ALIBABA_CLOUD_ACCESS_KEY_ID")
    secret_key = get_env("ALIBABA_CLOUD_ACCESS_KEY_SECRET")
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<-EOF
    terraform {
      required_providers {
        alicloud = {
          source  = "aliyun/alicloud"
          version = "1.258.0"
        }
        archive = {
          source  = "hashicorp/archive"
          version = "2.7.1"
        }
      }
    }

    provider "alicloud" {
      region = "ap-northeast-1"
    }
  EOF
}
```

先ほど控えた Alibaba Cloud のアクセスキーとシークレットキーを `ALIBABA_CLOUD_ACCESS_KEY_ID` および `ALIBABA_CLOUD_ACCESS_KEY_SECRET` 環境変数として Terragrunt に渡します。
DRY にするために generate ブロックを使って provider を定義します。東京リージョンをデフォルトにしたいので `ap-northeast-1` を指定します。

また、ロック用にキーバリューストア Tablestore (OTS) を利用することもできますが、今回は見送りました。Tablestore には Public / VPC / Classic (Internal) の http エンドポイントがあり、ローカル PC からの接続なら通常 Public を利用します。しかし Network ACL を Public 経路に挟み込むことができないので、全世界からアクセス可能でセキュリティ的にリスクを抱えることになります。
ローカル PC から安全に利用するためには VPN や専用線を通して VPC エンドポイントを経由して Tablestore にアクセスするべきですが、基本的に単独作業で開発するためロック機構の重要性が薄く、VPN 運用のコストを払ってまでロック機構を導入するモチベがありませんでした。
実務で運用する際は ECS Instance を立てて、VPC エンドポイントからロック用 DB にアクセスすることになるでしょうね。

今回コンソールを通じて backend OSS を作成しましたが、実は `terraform-alicloud-modules/remote-backend/alicloud` モジュールを利用することでロック用 OTS とまとめて backend を一発で構築することができます。[公式ドキュメント](https://www.alibabacloud.com/help/en/terraform/five-minute-introduction-to-alibaba-cloud-terraform-oss-backend)に詳細が記載されていますが、以下の記事も参考になりますね。

https://zenn.dev/kaikakin/articles/8e0b1ea308b00a

`terragrunt.hcl` はこんな感じです。`root.hcl` と LINE Messaging API の環境変数をここで読み込ませます。
また `service` ディレクトリ配下を参照する際に必要な `root_path` も定義します。`path.module` 等を利用すると terragrunt-cache ディレクトリ内での参照になり `root.hcl` が存在する `infra` 階層以上に自然にアクセスできないので、hcl 内で予めプロジェクトルートへの絶対パスを取得しておきます。

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "${get_parent_terragrunt_dir("root")}/modules/bot"
}

inputs = {
  line_channel_secret = get_env("LINE_CHANNEL_SECRET")
  line_access_token   = get_env("LINE_ACCESS_TOKEN")
  dashscope_api_key   = get_env("ALIBABA_DASHSCOPE_API_KEY")
  root_path           = get_repo_root()
}
```

`terragrunt init/plan` して成功すれば一旦終了です。`modules/bot` 配下については次の項で深掘っていきます。

## Terragrunt で FC のインフラを構築

modules の構成は以下の通りです。一つずつ解説していきます。

```
infra/modules/bot/
├── fc.tf
├── artifacts.tf
├── ram.tf
├── logs.tf
└── variables.tf
```

### fc.tf（Function Compute 本体）

```hcl
resource "alicloud_fcv3_function" "this" {
  function_name = "linebot-fc"
  runtime       = "nodejs20"
  handler       = "index.handler"
  cpu           = 0.25
  memory_size   = 256
  disk_size     = 512

  environment_variables = {
    LINE_CHANNEL_SECRET = var.line_channel_secret
    LINE_ACCESS_TOKEN   = var.line_access_token
    DASHSCOPE_API_KEY   = var.dashscope_api_key
  }

  role = alicloud_ram_role.fc_log_role.arn

  log_config {
    project  = alicloud_log_project.fc_logs.project_name
    logstore = alicloud_log_store.fc_logs.logstore_name
  }

  code {
    oss_bucket_name = alicloud_oss_bucket.artifacts.bucket
    oss_object_name = alicloud_oss_bucket_object.zipfile.key
  }
}

resource "alicloud_fcv3_trigger" "http" {
  trigger_name = "http"
  trigger_type = "http"
  qualifier    = "LATEST"
  trigger_config = jsonencode({
    disableURLInternet = false,
    authType           = "anonymous",
    methods            = ["POST"]
  })
  function_name = alicloud_fcv3_function.this.function_name
}

# output.tf に分離しても良い
output "fc_http_public_url" {
  value = alicloud_fcv3_trigger.http.http_trigger[0].url_internet
}
```

Function Compute 本体を定義しています。
FC には複数の世代がありますが、今回は最新世代の V3 を利用しています。

AWS Lambda 同様、ソースコードと依存ライブラリは zip ファイルにまとめてアップロードします。
ファイルサイズの都合上、`code.zip_file` でローカルからアップロードするのではなく、OSS 上の zip を参照する方針にしています。

FC には HTTP Trigger 機能があり、API Gateway を経由せずに直接 Web API として公開できます。alicloud_fcv3_trigger を定義することでこの仕組みを利用し、生成された URL を output として参照できるようにしました。

また、Web API 以外にも EventBridge や タイマー（Scheduled Trigger） を使ったイベント駆動の実行が可能らしいです。

### artifacts.tf（FC を構成する zip ファイルの生成と更新）

```hcl
resource "alicloud_oss_bucket" "artifacts" {
  bucket = "linebot-oss-bucket-fc-artifacts"
}

resource "alicloud_oss_bucket_object" "zipfile" {
  bucket = alicloud_oss_bucket.artifacts.bucket
  key    = "linebot-oss-object-fc-artifacts-${substr(data.archive_file.zip.output_sha, 0, 12)}.zip"
  source = data.archive_file.zip.output_path
}

data "archive_file" "zip" {
  type        = "zip"
  source_dir  = "${var.root_path}/services/bot"
  output_path = "${path.module}/code.zip"
}
```

`artifacts.tf` では、FC で利用するコードと依存ライブラリをまとめた zip ファイルの処理を行なっています。

小規模なファイルはローカルからアップロードすることが可能ですが、今回のように一定のサイズを超える zip ファイルに関しては OSS を経由して FC で参照する形式を取ることが要求されます。

現在時点で Alicloud provider には AWS provider における `source_code_hash` に相当する機能がなく、OSS にアップロードした zip ファイルの更新を FC 側で反映しないので、オブジェクト名を変更することで擬似的に変更を検知させるようにしています。

また、圧縮操作に関しては `hashicorp/archive` の zip ユーティリティを利用しています。プラットフォーム間の一貫性のためにプロバイダを利用していますが、プロジェクトの制約等で追加できない場合は `null_resource` 等でカスタムコマンドを実装したり手で実施することになると思います。

### ram.tf（FC の権限設定）

```hcl
resource "alicloud_ram_role" "fc_log_role" {
  role_name   = "linebot-role-fc-log-writer"
  description = "Allow FC to write logs to SLS"
  assume_role_policy_document = jsonencode({
    Version = "1",
    Statement = [{
      Effect    = "Allow",
      Action    = "sts:AssumeRole",
      Principal = { Service = ["fc.aliyuncs.com"] }
    }]
  })
}

resource "alicloud_ram_role_policy_attachment" "fc_log_role_attach" {
  role_name   = alicloud_ram_role.fc_log_role.role_name
  policy_type = "System"
  policy_name = "AliyunLogFullAccess"
}
```

`ram.tf` では FC の権限設定を実装しています。
AWS と同様、ポリシーを作成してロールにアタッチし、Assume Role で権限を付与しています。
今回は SLS 関係のマネージドポリシーを利用して手抜きしましたが、実務上ではポリシーも細かく定義することになるのではないでしょうかね。

### logs.tf（ログストリームの作成）

```hcl
resource "alicloud_log_project" "fc_logs" {
  project_name = "linebot-sls-pj-fc-logs"
  description  = "Function Compute logs"
}

resource "alicloud_log_store" "fc_logs" {
  project_name     = alicloud_log_project.fc_logs.project_name
  logstore_name    = "linebot-sls-ls-fc-logs"
  retention_period = 90
}

resource "alicloud_log_store_index" "fc_logs_index" {
  project  = alicloud_log_project.fc_logs.project_name
  logstore = alicloud_log_store.fc_logs.logstore_name

  full_text {
    case_sensitive = false
  }
}
```

FC のデバッグ、ログ出力のためログストリームを構築しています。

AWS CloudWatch Logs とは異なり、Alibaba Cloud では Log Project と Log Store の 2 階層を指定してログが出力されています。
ただ、これらだけの指定だとコンソール上でログが表示されないため、Log Store Index も指定してログを有効化する必要があります。

## Messaging API と Qwen を FC で連携する

最後に Function Compute の内容を service 配下に記述します。ディレクトリ構成は以下です。
Qwen LLM のインターフェース、LINE API 関連のユーティリティ関数、Bot の処理ルーチンをそれぞれ `llm`, `line`, `bot` にまとめました。エントリポイントは `index.js` であり、ここから各関数を呼び出していきます。

```
service/bot/
├── src/
│   ├── llm/
│   │   └── openaiClient.js
│   ├── line/
│   │   ├── client.js
│   │   ├── isMentionToBot.js
│   │   └── verifySignature.js
│   └── bot/
│       └── process.js
├── index.js
└── package.json
```

### line directory

`client.js`

```javascript
async function replyMessage(replyToken, text) {
  const res = await fetch("https://api.line.me/v2/bot/message/reply", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${ACCESS_TOKEN}`,
    },
    body: JSON.stringify({
      replyToken,
      messages: [{ type: "text", text }]
    })
  });

  if (!res.ok) {
    const body = await res.text();
    throw new Error(`LINE API ${res.status}: ${body}`);
  }
}
```

LINE でメッセージを投稿する際は reply エンドポイントに JSON を POST すれば良いですね。

`isMentionToBot.js`

```javascript
async function isMentionToBot(ev) {
  const mentionees = ev?.message?.mention?.mentionees;
  if (!Array.isArray(mentionees) || mentionees.length === 0) return false;

  // bot 宛てメンションか？
  if (mentionees.some(m => m?.isSelf === true)) return true;

  // @all 許可
  if (mentionees.some(m => m?.type === 'all')) return false;

  return false;
}
```

`isSelf` 属性という便利な属性があるので、bot 自身がメンションされたことを確認可能です。グループチャットで使用する際に重宝します。

`verifySignature.js`

```javascript
import crypto from 'node:crypto';

const CHANNEL_SECRET = process.env.LINE_CHANNEL_SECRET;

// LINE Webhook 署名検証
function verifyLineSignature(bodyStr, headers) {
  const signature = headers['X-Line-Signature'];

  const hmac = crypto.createHmac('sha256', CHANNEL_SECRET);
  hmac.update(bodyStr);
  const expected = hmac.digest('base64');

  return signature === expected;
}
```

API は外部に露出しているため、シークレット情報で署名検証を行い不審な POST の処理を回避します。これで外部からの干渉を阻止できます。

### llm directory

`openaiClient.js`

```javascript
import OpenAI from 'openai';

const API_KEY = process.env.DASHSCOPE_API_KEY;

const openai = new OpenAI({
  apiKey: API_KEY,
  baseURL: "https://dashscope-intl.aliyuncs.com/compatible-mode/v1"
});

async function chatQwenTurbo(userText) {
  const completion = await openai.chat.completions.create({
    model: "qwen-turbo",
    messages: [
      {
        role: "system",
        content: "あなたはチャット相手です。名前は\"わんわんお\"です。必ず日本語で返答してください。返答はとても短く（1〜2文）、会話内容とあまり関係ない単語をときどき混ぜ込みます。"
      },
      { role: "user", content: userText || "こんにちは" }
    ],
    stream: false,
    enable_thinking: false
  });

  return completion.choices?.[0]?.message?.content ?? "（応答なし）";
}
```

Qwen は openai ライブラリ互換のインターフェースで呼び出すことができます。
Streaming を false にした時、思考モードが使えなかったので `enable_thinking: false` にすることに注意。Thinking を利用したい場合はコードが複雑化しそうです。

### bot directory and index.js

あとは今までのロジックをまとめるだけです。読めばわかるのでこちらに関しては説明しません。

`process.js`

```javascript
import { replyMessage } from '../line/client.js';
import { chatQwenTurbo } from '../llm/openaiClient.js';
import { isMentionToBot } from '../line/isMentionToBot.js';

async function process(ev) {
  const replyToken = ev?.replyToken;
  const message    = ev?.message?.text;

  // メンション時のみ反応
  const shouldReply = await isMentionToBot(ev);
  if (!shouldReply) return;

  let reply;
  try {
    reply = await chatQwenTurbo(message);
  } catch (e) {
    console.error("Qwen API error:", e);
    reply = "LLM呼び出し中にエラーが発生しました";
  }

  await replyMessage(replyToken, reply);
}
```

`index.js`

```javascript
import { verifyLineSignature } from './src/line/verifySignature.js';
import { process } from './src/bot/process.js';

// --- レスポンスヘルパ ---
function res(statusCode, body, headers = { "Content-Type": "text/plain" }) {
  return { statusCode, headers, body };
}

const ok        = (body = "ok")                => res(200, body);
const badReq    = (body = "bad request")       => res(400, body);
const forbidden = (body = "invalid signature") => res(403, body);
const error     = (body = "error")             => res(500, body);

// --- ハンドラ ---
export async function handler(event, context, callback) {
  let outer;
  try {
    const str = event.toString('utf8');
    outer = JSON.parse(str);
  } catch {
    return callback(null, badReq());
  }

  try {
    // --- 署名検証 ---
    const bodyStr = typeof outer.body === "string" ? outer.body : JSON.stringify(outer.body || {});
    if (!verifyLineSignature(bodyStr, outer.headers)) {
      console.warn("Signature mismatch");
      return callback(null, forbidden());
    }

    // --- 本処理 ---
    const body = JSON.parse(bodyStr);
    const ev = body?.events?.[0] ?? null;

    if (ev?.type === "message" && ev.message?.type === "text") {
      await process(ev);
    }

    return callback(null, ok());
  } catch (e) {
    console.error(e);
    return callback(null, error());
  }
};

```

以上で FC のロジックは完了です！

## 動作確認

![](/images/create-line-chatbot-with-alicloud-and-qwen/operation-check.png)

うまくいってそうです。

## おわりに

閲覧ありがとうございます。
VPC 内の ECS インスタンスでの Terragrunt 実行が強く要求されている感があり、中々ローカル PC でシステム構築するのは難しい印象でしたね。

今回は一部コンソールを利用してリソース操作を行なっておりましたが、CLI から同様の操作を行える [Aliyun CLI](https://github.com/aliyun/aliyun-cli) も公開されています。Aliyun CLI の導入に関してはこちらの記事が詳しいので興味ある方はぜひ参考にしてください。

https://qiita.com/n_watanabe/items/1ca89c3a3cf5dc650305