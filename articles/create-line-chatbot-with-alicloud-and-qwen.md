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

今回は Alibaba Cloud を利用して LINE 上で bot を作成する手順についてご紹介します。
皆様におかれましては、コンプライアンスとセキュリティの観点から基本的に業務上は AWS, Azure, GCP の 3 択から選ぶことになると思います。
一方で Alibaba Cloud は世界第四位の Cloud Infrastructure にもかかわらず、業務上で使われることは中国市場案件ではない限りなかなかお目にかかる機会はないと思います。
今回は、Alibaba Cloud を LINE 上で動かす bot を作成して、Alibaba Cloud に対する免疫をつけることを目的としてプロジェクトを始めました。
なお、会社の業務とは完全に別個にするために、自分で alibaba cloud account を作成して個人用の LINE を利用していることにご注意してください。会社とは全く関係がありません。
筆者が AWS を利用している関係で、ところどころ AWS サービスとの比較が出てきます。

AWS との対応は以下の通りです。覚えましょう。
- RAM (Resource Access Management) <-> IAM
- OSS (Object Simple Storage) <-> S3
- Function Compute <-> Lambda
- SLS (Simple Log Service) <-> CloudWatch Logs
- Tablestore <-> DynamoDB

## LINE Messaging API を用意する

LINE で bot を作成する際に利用するのが LINE Messaging API です。
Messaging API を利用することで個人チャットや所属しているグループチャットの情報を特定の API に POST して処理したり、LINE にメッセージ等を投下したりすることができます。
本アプリを作成する際には二段階のプロセスを踏むことになると思います。
- LINE bot を作成する
- チャネルを作成し、Messaging API を有効化する


## Alibaba Cloud のアカウントを用意する

[Alibaba Cloud](https://www.alibabacloud.com/ja?_p_lc=1) にアクセスします。右上のログインボタンからログインページに飛び、右下付近にある登録リンクから登録ページに飛びましょう。
登録ボタンを押すと、メールアドレスとパスワードが要求されます。その後画面の指示に従って住所やクレジットカード情報を埋めていくとサインアップが完了します。こちらの流れに関しては、基本的にウィザードに従えばいいので詳しくは解説しません。

Alibaba Cloud に登録すると各種リソースの無料枠が付いてきます。今回のアプリでは基本的に利用しませんが、今後インスタンスを立てたり検証する際は積極的に使うと良いと思います。

Alibaba 限らず Cloud サービスを個人運用する際にはまず budget を設定しましょう。これは費やしたコストを追跡して予め立てた閾値を超えた場合に指定アドレスに警告メールを送付するシステムです。
請求されてからでは遅いので早めにやっておきましょう。

AWS などと違い、Alibaba Cloud ではアカウントを作成したら即時で各種サービスが使えるというわけではありません。一部サービスは即時で使うことができますが、いくつかのサービスは購入手続きが必要です。
今回利用するサービスで有効化が必要なのは Simple Log Service (SLS) と Object Simple Storage (OSS) です。これらを有効化しない限り terraform 等で CLI から作成することはできません。

## Terragrunt (Terraform) 用の backend を用意する

Terragrunt で Alibaba Cloud のリソースを操作する際に必要なユーザと tfstate 保存用の bucket の整備を行います。
CLI のためにルートユーザでアクセストークンを発行するわけにはいかないのと、Alibaba Cloud の調査を兼ねて Web UI での操作を行います。

まず、RAM を利用してユーザの作成と権限付与を行います。ユーザ等のエンティティと権限を管理する Alibaba cloud の機能を RAM (Resource Access Management) と呼び、AWS の IAM に相当します。

Management Console から RAM を開いてユーザを作成します。ユーザの作成ボタンを押すと以下のような画面が表示されると思います。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-create.png)

Using permanent AccessKey to access を ON にして他のオプションをデフォルトにします。ログイン名と表示名を記載して OK ボタンを押すと、以下のような画面が表示されます。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-create-credentials.png)

CSV File をダウンロードして控えておきましょう。ここに書いてある AccessKey ID と AccessKey Secret を認証情報として terraform がリソースを作成します。

ユーザを作成したら、次は権限付与を行います。権限管理タブを開き、Grant Permission ボタンをクリックしてください。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant.png)

モーダルが開きます。今回は RAM 関連の操作も行うので、AdministratorAccess のマネージドポリシーを選択し付与します。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant-detail.png)

AdministratorAccess のトグルを ON にしようとすると警告が出ます。
今回は試用なので強い権限を与えていますが、RAM を操作しなければ警告通りに PowerUserAccess に変更するべきですし、不必要な機能を操作できないよう最小権限を与えるべきではあります。

![](/images/create-line-chatbot-with-alicloud-and-qwen/ram-user-grant-warning.png)

tfstate 保存先の bucket を作成します。
Management Console から OSS を開き、バケット一覧画面を表示しましょう。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-create-bucket.png)

バケットの作成をクリックします。細かい設定は置いといて、公開アクセス禁止の状態で日本リージョンに作成します。

![](/images/create-line-chatbot-with-alicloud-and-qwen/oss-create-bucket-detail-3.png)

次は Terragrunt の設定をしましょう。ローカルのレポジトリを作成し、以下のような構成でファイルツリーを構成します。

- infra
  - live / dev / bot
    - terragrunt.hcl
  - modules / bot
    - .gitkeep
  - root.hcl
- services
  - ...

`root.hcl` の中に provider および backend 設定を記述します。

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

先ほど CSV として控えた Alibaba Cloud のアクセスキーとシークレットキーを `ALIBABA_CLOUD_ACCESS_KEY_ID` および `ALIBABA_CLOUD_ACCESS_KEY_SECRET` 環境変数として Terraform に渡します。
DRY にするために generate ブロックを使って provider を定義します。中国リージョンにすると現地の法律が色々厄介なため `ap-northeast-1` を指定し、デフォルトを東京リージョンにします。

今回はロック用のキーバリューストアは利用しません。
Alibaba Cloud には DynamoDB に変わる NoSQL DB として Tablestore が存在し、これを Terraform のロック機構に利用することができますが、セキュリティ上の懸念点があります。

Tablestore は基本的に https エンドポイントを経由してデータの操作を行います。
エンドポイントは internet, internal, vpc の三種類ありますが、今回自分が用意した作業環境はローカルの PC で terragrunt を実行する感じなので、URL を知っていれば誰でもアクセスできる internet を選択せざるを得ません。インスタンスを作成して VPC で作業するのは工数と費用の面から避けました。
Internet endpoint に関しては WAF を導入して IP で弾くこともできますが、基本的に一人で作業するのでロックが必要とされる場面がなく、プロキシ立ててまでロック機構を導入するモチベがありませんでした。

今回 GUI を通じて backend OSS を作成しましたが、実は `terraform-alicloud-modules/remote-backend/alicloud` モジュールを利用することでロック用 tablestore とまとめて構築することができます。今回は management console の調査を兼ねていたので Web UI を利用していましたが、実際に作成する際はこちらも利用すると良いと思います。

https://www.alibabacloud.com/help/en/terraform/five-minute-introduction-to-alibaba-cloud-terraform-oss-backend

https://zenn.dev/kaikakin/articles/8e0b1ea308b00a

`terragrunt.hcl` はこんな感じです。`root.hcl` と LINE Messaging API の環境変数をここで読み込ませます。
また `service/` ディレクトリ配下を参照する際に必要な `root_path` も定義します。`path.module` 等を利用すると terragrunt-cache ディレクトリ内での参照になり `root.hcl` が存在する `infra` 階層以上に自然にアクセスできないので、hcl 内で予めプロジェクトルートへの絶対パスを取得しておきます。

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

`terragrunt init` して成功すれば一旦終了です。`modules/bot` 配下については次の項で深掘っていきます。

## Terragrunt で Function Compute (FC) のインフラを構築する

module 構成は以下の通り、一つずつ解説していきます。

- infra / modules / bot
    - artifacts.tf
    - fc.tf
    - logs.tf
    - ram.tf
    - output.tf
    - variables.tf

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
```

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

### 他

`output.tf`

```hcl
output "fc_http_public_url" {
  value = alicloud_fcv3_trigger.http.http_trigger[0].url_internet
}
```

## Qwen の有効化

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