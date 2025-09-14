---
title: "Alibaba cloud と Qwen LLM で LINE Chatbot を作成する"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["alibaba", "alibabacloud", "terraform", "terragrunt", "qwen"]
published: false
---

## この記事で得られること

- LINE ChatBot を作成する技術

## はじめに

こんにちは、今回は世界第四位の市場シェアにも関わらず、業務上では殆どお目にかかれないクラウド、Alibaba Cloud で遊びたくなったので LINE 上で Qwen を使ったチャットボットを作成してみました。

コンプライアンス等の要請から会社の業務用ツールとは完全に別個にしたいのと、友人間の流れみたいなところもあったのでプライベートでアカウントを取得して、LINE でボットを作ってみることにしました。

地政学的な問題から避けられがちな Alibaba Cloud ですが、実は世界 21 ヶ国、63 箇所のアベイラビリティゾーンを展開しており、中国・ASEAN とのビジネスを行う上では選択肢の第一候補として挙がってくるものだと認識しています。
喫緊で必要になるものではないですが、将来性も踏まえて手をつけておきたいと思ったりしていますね。

筆者が主に AWS を使っている関係上、AWS との比較が出てくると思います。鬱陶しく感じる方もいらっしゃるとは思いますが、ご了承願います。

## LINE Messaging API を用意する

LINE bot は LINE Messaging API を利用して設定します。
Messaging API を利用することで個人チャットや所属しているグループチャットの情報を Web API 経由で受信したりメッセージを送信したりすることができます。
詳しくは以下の記事を参考にしてください。

https://zenn.dev/kou_pg_0131/articles/line-push-text-message

公式ドキュメントはこちらです。必要に応じて参照してください。

https://developers.line.biz/ja/docs/messaging-api/building-bot/

Bot 名は「わんわんお」にしました。

## Alibaba Cloud のアカウントを用意する

[Alibaba Cloud](https://www.alibabacloud.com/ja?_p_lc=1) にアクセスします。右上のログインをクリックした後に登録リンクを踏んでサインアップを始めます。
画面の指示に従って住所やクレカ情報を埋めていき、サインアップを完了します。基本的にウィザードに従えばいいので詳しくは説明しません。

Alibaba Cloud に登録すると各種リソースの無料枠が付与されます。
今回は積極的に利用しないですが、今後インスタンスを立てたり検証する際に使ってみようと思います。

また、Alibaba Cloud では一部サービスは購入手続きを踏まないと利用できないようです。例えば terraform で有効化を行わないままリソースを apply しようとすると以下のようなエラーが発生します。Web UI で必ず有効化手続きを行いましょう。

```
* Failed to execute "terraform apply" in ./.terragrunt-cache/32vclavby77fAy-C9VVfRTiUmRA/dP5CvTElUG-Yrd5DSd1xIqgSbAM
  ╷
  │ Error: [ERROR] terraform-provider-alicloud/alicloud/resource_alicloud_log_project.go:114: Resource alicloud_log_project / Failed!!! [SDK alibaba-cloud-sdk-go ERROR]:
  │ SDKError:
  │    StatusCode: 401
  │    Code: InvalidAccessKeyId
  │    Message: Your SLS service has not been opened.
  │    Data: {"httpCode":401,"requestId":"68BF6BB66B4839897F388201","statusCode":401}
  │ 
  │ 
  │   with alicloud_log_project.fc,
  │   on ram.tf line 1, in resource "alicloud_log_project" "fc":
  │    1: resource "alicloud_log_project" "fc" {
  │ 
  ╵
```

今回利用するサービスで有効化が必要なのは Simple Log Service (SLS) と Object Simple Storage (OSS) です。

Management Console から OSS を開くと、アクティブ化要求画面が現れます。「今すぐ有効にする」をクリックして画面の指示に従い、購入手続きを行います。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-info-activation.png)

SLS に関してもコンソールから開くとアクティブ化の要求画面が現れます。「Log Service のアクティブ化」をクリックして購入手続きを行なってください。
途中で以下のような謎のモーダルが出現するので、URL をブラウザのアドレスバーにコピペしてアクセス。

![](/images/create-line-chatbot-with-alicloud-and-qwen/sls-purchase-billing-url.png)

すると次のような検証手続きの画面に映るので、画面の指示に従っていくと購入手続きが完了です。

![](/images/create-line-chatbot-with-alicloud-and-qwen/sls-purchase-billing-check.png)

最後に、Alibaba 限らず Cloud サービスを個人運用する際にはまず budget を設定しましょう。これは費やしたコストを追跡して予め立てた閾値を超えた場合に指定アドレスに警告メールを送付するシステムです。
請求されてからでは遅いので早めにやっておきましょう。

## Qwen の有効化

Alibaba Cloud がホストしている LLM, Qwen についても有効化しておきます。

[Alibaba Cloud Model Studio](https://modelstudio.console.alibabacloud.com/) にアクセスします。ここから Qwen や画像生成モデル Wan などのモデルを管理するダッシュボードを利用することができます。
Model Studio は中国版と国際版に分かれているため、アクセス後右上に International Edition と記載されていることを確認してください。

Model Studio は無償試用制度があり、一定の条件下で各種モデルを無料で呼び出すことが可能です。ログイン後右上にある `New User Offer ... Activate Now` をクリックして利用を開始します。詳しくは[ドキュメント](https://modelstudio.console.alibabacloud.com/?tab=doc#/doc/?type=model&url=2766612)を参照してください。

次に右上の `Get API Key` から Secret Key を取得します。クリックすると Key の一覧ページに遷移するので `Create API Key` から Default workspace に鍵を一つ追加しました。以下のようにエントリが追加されたので、API Key をコピーして控えます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/qwen-dashboard-apikey.png)

そのまま右の Workspaces をクリックしてワークスペースの一覧ページに遷移し Authorization & Throttling Settings を確認。

![](/images/create-line-chatbot-with-alicloud-and-qwen/qwen-dashboard-workspaces.png)

クリックすると利用可能なモデルの一覧が現れるので、その中から使いたいモデルの名前を選んでおきます。

## Terragrunt 用の backend を用意する

Terragrunt で Alibaba Cloud のリソースを操作するためのユーザと tfstate 用の OSS bucket の整備を行います。
今回は Alibaba Cloud の調査も兼ねているので Web UI でリソースを作成します。

ユーザの作成と権限付与は RAM (Resource Access Management) で行います。RAM はユーザ等のエンティティと権限を管理する機能であり、AWS IAM に相当します。

まず、Management Console から RAM を開いてユーザを作成します。ユーザの作成ボタンを押すと以下のような画面が表示されます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-create.png)

Using permanent AccessKey to access を ON にして他のオプションをデフォルトにします。ログイン名と表示名を記載して OK ボタンを押すと、以下のような画面が表示されます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-create-credentials.png)

CSV File をダウンロードして控えておきましょう。ここに書いてある AccessKey ID と AccessKey Secret を認証情報として terraform を実行します。

ユーザを作成したら、次は権限付与を行います。権限管理タブを開き、Grant Permission ボタンをクリックしてください。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant.png)

モーダルが開きます。今回は RAM 関連の操作も行うので、AdministratorAccess のマネージドポリシーを選択し付与します。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant-detail.png)

AdministratorAccess のトグルを ON にしようとすると警告が出ますが、RAM を terragrunt で管理することに決めているのでこのままでいきます。RAM を管理しないのであれば警告通り PowerUserAccess に変更すべきですね。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant-warning.png)

次に tfstate 保存先の OSS bucket を作成します。
Management Console から OSS を開き、バケット一覧画面を表示しましょう。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-create-bucket.png)

バケットの作成をクリックします。細かい設定は置いといて、公開アクセス禁止の状態で東京リージョンに作成します。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-create-bucket-detail-3.png)

Web UI での準備が完了したので次は Terragrunt の設定です。以下のような構成でファイルツリーを構成します。

- infra
  - live / dev / bot
    - terragrunt.hcl
  - modules / bot
    - .gitkeep
  - root.hcl
- services
  - ...

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
DRY にするために generate ブロックを使って provider を定義します。東京リージョンをデフォルトにするために `ap-northeast-1` を指定します。

また、ロック用にキーバリューストア OTS (Object Table Store) を利用することもできますが、今回は利用しないことにしました。
OTS は基本的に http エンドポイントを経由してトランザクションを受け付けますが、ローカルの PC からだと Internet, Internal, VPC endpoint の三種類の内、利用できるのは Internet endpoint のみになります。
Internet endpoint を安全に利用するためには WAF を導入して IP で弾く機構を実装しないといけませんが、基本的に一人で作業するのでロックの必要性が小さく、プロキシ立ててロック機構を導入するモチベがありませんでした。
VPC 内で作業したり CI/CD を構築する際はロック用 DB を導入することになるでしょうね。

今回 GUI を通じて backend OSS を作成しましたが、実は `terraform-alicloud-modules/remote-backend/alicloud` モジュールを利用することでロック用 OTS とまとめて backend を一発で構築することができます。必要に応じてこちらも利用すると良いと思います。

https://www.alibabacloud.com/help/en/terraform/five-minute-introduction-to-alibaba-cloud-terraform-oss-backend

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

## Terragrunt で Function Compute (FC) のインフラを構築する

module 構成は以下の通り、一つずつ解説していきます。

- infra / modules / bot
    - artifacts.tf
    - fc.tf
    - logs.tf
    - ram.tf
    - variables.tf

### fc.tf

`fc.tf`

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
Compute Instance は CPU, Memory, Disk を数値として定義する仕組みで、runtime は node.js, Java, Python, PHP, Go, .NET を組み込みで装備しているとのこと。今回は node.js を利用して実装するが、現在 AWS で利用できる Node 24.x は Alicloud では未対応とのこと、最新 ver を使いたい方はカスタムランタイムを作成することになると思います。

AWS Lambda 同様、ソースコードと依存ライブラリは zip ファイルにまとめてアップロードする。
今回は OSS 上の zip を参照するが、`code.lambda_zip` 属性でローカルの zip ファイルを上げることも可能。

LINE Messaging API は http エンドポイントを通して情報を授受するので、Web API を通じて FC が発火されるように設定した。Web API 以外にも EventBridge (Alicloud) や時間指定も可能とのこと。
Amazon API gateway よりも、AWS Lambda Function URLs の方が近い。

### artifacts.tf

`artifacts.tf`

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

FC で利用するコードと依存ライブラリをまとめた zip ファイルの処理を行っている。
小さいファイルはローカルからアップロードすることが可能だが、xx KB を超える zip ファイルに関しては OSS に一度あげてから FC で参照する形式を取ることが要求される。依存ライブラリの `openai` が大きかったため、OSS 経由のアップロードで実装した。
今回は `hashicorp/archive` で提供されている zip ユーティリティを利用して圧縮ファイルを作成したが、プロジェクト制約等で追加できない場合は null_resource 等で代替したり手で作成することも可能。ただし、プラットフォームに依りそう。

### ram.tf

`ram.tf`

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

FC の権限を定義している。
AWS と同様、ポリシーを作成してロールにアタッチし、先の FC 定義でロールを Assume Role する。
今回は時間の都合上マネージドポリシーを利用したが、実務上ではポリシーも細かく定義することになるだろう。

### logs.tf

`logs.tf`

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

FC のデバッグ、ログ出力のためログストリームを構築する。
AWS の CloudWatch Logs とは異なり、Alibaba Cloud では Log Project と Log Store の 2 階層を指定してログが出力される。
ただし、これらだけの指定だと Web UI 上でログが表示されないため、Log Store Index も指定してログの有効化もする必要がある。

## Messaging API と Qwen を FC で連携する

Function Compute の内容は service 配下に記述しています。ディレクトリ構成は以下です。

- service / bot
  - src
    - line
      - openaiClient.js
    - llm
      - client.js
      - isMentionToBot.js
      - verifySignature.js
    - bot
      - process.js
  - index.js
  - package.json

一つずつ解説してきます

### line Directory

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

LINE での message の投稿は reply エンドポイントに JSON を POST すれば良い。

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

isSelf 属性という便利な属性があるので、bot 自身がメンションされたことを確認可能。
グループチャットで使用する際に利用する。

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

API は外部に露出しているので、シークレットで署名検証を行い不審な POST の処理を回避する。

### llm

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

Qwen の API は OpenAI のライブラリから呼び出すことができる。
Streaming を false にした時、思考モードが使えないので `enable_thinking: false` にすることを忘れない。

### bot

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

あとは今までのロジックをまとめるだけ。

### エントリポイント

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

## おわりに

今回は一部 management console を利用してリソース操作を行なっておりましたが、CLI から同様の操作を行える [Aliyun CLI](https://github.com/aliyun/aliyun-cli) も公開されています。Aliyun CLI の導入に関してはこちらの記事が詳しいので興味ある方はぜひ参考にしてください。

https://qiita.com/n_watanabe/items/1ca89c3a3cf5dc650305