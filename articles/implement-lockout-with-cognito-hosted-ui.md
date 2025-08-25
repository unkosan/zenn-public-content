---
title: "Cognito Hosted UI 上でロックアウトをカスタム制御する"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cognito", "terraform"]
published: true
---

## はじめに
社内向けの小規模アプリケーションを作成する際に、Cognito の Hosted UI を利用して簡易的に認証基盤を構築することは多いはず。
Hosted UI を利用すれば自前で Cognito の API を叩くカスタムページを作成せずに済むので確かに便利ですよね。

ただ、実際の案件上では企業のパスワードポリシーが存在して、これに準拠するように制御をカスタマイズしなければならないこともある。
英数字記号混合や桁数、MFA などは Cognito がネイティブに実装しているため簡単に調節可能だが、ロックアウト機構に関しては残念ながら標準で搭載されている機構のパラメータを自由に調節できないようになっている。

以下 AWS developer guide から引用。将来的に変更される可能性があるとは書いてあるが、現状では 15 分以上ロックアウトの要求がある場合は追加実装が必要になる。

> ユーザーのパスワードによるサインイン試行が 5 回失敗すると、認証されていない API オペレーションと IAM 認可の API オペレーションのどちらでリクエストされたかにかかわらず、Amazon Cognito はユーザーを 1 秒間ロックアウトします。ロックアウトの期間は、試行が 1 回失敗するたびに 2 倍になり、最大で約 15 分になります。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/authentication.html

今回、Cognito Hosted UI を利用する環境下でロックアウトのパスワードポリシーを準拠させる機会をいただいたが、関連する記事や情報が少なく苦労したので、現状で考えられる最善手を書き留めておく。
読者のヒントになれば幸いです。

以下、Authenticator App での MFA 認証を含む前提で話を進める。
Password 認証だけの話でも大体一緒なので適宜ご自身で補完しながら読んでいただければ結構です。

## 結論

基本的な方向性としては、Password 認証で得られたログから失敗回数を読み取り DynamoDB に書き込んだのち、失敗数に応じて Pre Authentication でロックアウトを実施するという方針。
ただログの配送までにラグがあるので flag を立てて処理中は一時的にロックアウトさせる。

- 必要コスト
  - Cognito Advanced Security Function (ASF) **PLUS**
  - DynamoDB, CloudWatch Logs, Lambda の使用料金
- 実装方針
  - DynamoDB: 失敗回数カウント用の DB. 
    - failCount: 失敗数カウンタ
    - lockUntil: ロックアウト期限
    - hasQueue: Password 認証成功・失敗から CloudWatch Logs にイベントが発行されるまで時間がかかる。その処理中であることを示すフラグ。
  - CloudWatch Logs
    - Cognito ASF で提供されるログの出力先
  - AuthGuard Lambda: ロックアウト処理を担当する
    - Cognito の Pre Authentication に紐づける
    - lockUntil より前の時刻か hasQueue が true なら落とす、そうでなければ set true して認証続行
  - FailCount Lambda: 失敗回数をカウントする。
    - CloudWatch Logs に subscription し、SingIn イベントでフィルタリングして発火。
    - `Password:Success` / `Password:Failure` が発行された時点で failCount を操作して hasQueue を set false. MFA 操作時には何もしない
- 実現される機能
  - パスワード失敗時、失敗カウントが更新されるまでユーザは一時ロックアウト
    - カウントが更新された際、即時にトライアル可能になる
    - 一時ロックアウトは数秒から数十秒にわたることがしばしばある。残念ながらどうしようもない。
  - パスワード成功かつ MFA 失敗時、失敗カウントは 0 になる。
    - MFA 失敗にはロックアウト、複数回の失敗を含む
  - パスワード成功かつ MFA 成功時、失敗カウントは 0 になりユーザはリソースにアクセスする
    - MFA 失敗後からの成功を含む
- 要注意点
  - パスワード成功が連続する時も初回の失敗時も、一時ロックアウトが発生するので UX がよくない。

## 実装方法の調査

Cognito からパスワード認証成功・情報を抜き出してカウントし、Pre Auth でカウンタを読み込んでアクセス遮断する方針で調査を進めていた。

https://dev.classmethod.jp/articles/how-to-check-the-cognito-authentication-log/

https://qiita.com/shinichi_yoshioka2/items/c1af96f83502a92c5998

上の記事によると、CloudTrial から成功・失敗の証跡をユーザ ID (`userSub`) を取得することは可能らしいと書かれているため、この証跡を EventBridge と連携して Lambda を紐付け DynamoDB に書き込むことを仮説として当初は考えていた。

しかし実装を進めていくうちに、紹介されている `InitiateAuth` はどうやら **Hosted UI からの認証イベントでは発行されない**らしいことが発覚。
`InitiateAuth` を実施するのは AWS SDK を通じた API 認証であり、Hosted UI ではプログラマティックな認証を適用できないとのこと。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/authentication.html?utm_source=chatgpt.com

Developer Guides に記載されているイベントを確認しても、成功・失敗時にトリガされるのは `Login_GET` 以外を除いて他になさそうな状況だった。
`Login_GET` を確認しても `userSub` / `userName` が存在せずユーザが特定できなかったため、有償プランである Advanced Security Function による実装を検討した。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/logging-using-cloudtrail.html

ASF を利用すると指定した CloudWatch Logs に `USER_AUTH_EVENTS` が吐き出される。
検討にあたって、ログの生成結果と条件を確認した。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/exporting-quotas-and-usage.html?utm_source=chatgpt.com

Developer Guides の内容に限界があったので、ログの生成条件を実際に確認すると、大体このような結果になった。

- **Password 通過・失敗時点で `$.message.challenges[]` に `Password:Success` / `Password:Failure` が追加されて発行**
- MFA 通過・失敗時点で `Mfa:Success` / `Mfa:Failure` が追加されて発行、失敗したら失敗した回数だけ追加されて発行
- MFA タイムアウト時は発行されない

Password 認証前に Pre Authentiation Lambda が起動することを踏まえ、結局 Password 通過・失敗時点のログだけ拾ってやればよいことになるので、`$.message.challenges[]`の末尾に `Password:Success` / `Password:Failure` が来たらカウンタを更新すれば良い、という感じで決着。`Mfa:*` 系が来た場合は放棄で OK.

## 実装

### DynamoDB

失敗カウンタと後述するロック機構を実現するために DB が必要。
今回は DynamoDB を選択したが、正直社内アプリレベルなら Cognito user pool の attribute に書き込んでも良い気がしている。
ただコスト面でスケールしないのでユーザ数が大きい場合は注意。

```hcl
resource "aws_dynamodb" "lockout" {
  name = "lockout-ddb"
  hash_key = "userSub"
  attribute {
    name = "userSub" # User ID, UUID で表示される
    type = "S"
  }
  # 他の属性
  # failCount (number) # 失敗回数
  # lockUntil (number) # ロックアウト期限
  # hasQueue (boolean) # カウント処理中フラグ
}
```

hasQueue に着目、これを使って認証終了から失敗回数の反映まで一時的なロックアウトを実施します。
Cognito から CloudWatch Logs に証跡が送出されるまでには数秒から数十秒ラグが発生しており、失敗回数が反映されるまでユーザは毎回ログインを拒否される。
正直 UX としては微妙だが、**このロック機構がないとカウント反映前に試行ができてしまう**ので、n 回の失敗で t 時間ロックアウトする要件を確実に守れなくなる。
もしこの一時ロックアウトを消したいのであれば Hosted UI を使わないという選択肢になるので、工数と要相談です。

### Cognito

Cognito user pool は大体こんな感じになる。

```hcl
resource "aws_cognito_user_pool" "this" {
  user_pool_tier = "PLUS"
  user_pool_add_ons {
    advanced_security_mode = "AUDIT"
  }
  lambda_config {
    pre_authentication = aws_lambda.authguard.arn
  }
}
resource "aws_cloudwatch_log_group" "userlog" {...}
resource "aws_cognito_log_delivery_configuration" "userlog" {
  user_pool_id = aws_cognito_user_pool.this.id
  log_configurations {
    event_source = "userAuthEvents"
    log_level    = "INFO"
    cloud_watch_logs_configuration {
      log_group_arn = aws_cloudwatch_log_group.userlog.arn
    }
  }
}
```

Plus プランは監査のみモードでログが出ます。フル機能にする必要ないです。
`aws_cloudwatch_log_group` は AWS provider v6.5.0 以上が必要な、かなり新しめの機能なので注意。

### Lambda (authguard)

ロックアウトを実現する Cognito Pre Authentication の Lambda を書くとしたらこんな感じ。

```javascript
const ddb = new DynamoDBClient({});
const tableName = "lockout-ddb";

exports.handler = async (event) => {
  const userSub = event?.request?.userAttributes?.sub;
  if (!userSub) {
    throw new Error("User ID not found.");
  }

  try {
    const res = await ddb.send(
      new GetItemCommand({
        TableName: tableName,
        Key: { userSub: { S: userSub }},
        ProjectionExpression: "lockUntil, hasQueue",
        ConsistentRead: true,
      })
    );
    const lockUntil = Number(res.Item?.lockUntil?.N || 0);
    const hasQueue = Boolean(res.Item?.hasQueue?.Bool);

    if (hasQueue) {
      // ログが出てこなくて hasQueue=true が永続化するのが怖ければ時間経過で
      // このブロックを無視する auto-healing 機能を記述してもよし
      throw new Error("Failure count is currently being updated");
    }

    const now = Math.floor(Date.now() / 1000);
    if (lockUntil > now) {
      throw new Error("This account is locked. Try again later.");
    }

    await ddb.send(
      new UpdateItemCommand({
        TableName: tableName,
        Key: { userSub: { S: userSub }},
        UpdateExpression: "SET hasQueue = :flag",
        ExpressionAttributeValues: {
          ":flag": { BOOL: true },
        }
      })
    );

    return event;
  } catch (err) {
    throw err;
  }
}
```

terraform 周りはこれだけ設定すれば良いと思います。
Lambda のリソースポリシーは Pre Auth 指定で自動付与されますが、関数を変更した時に付与されないことがあるので明示しておいた方が安全です。

```hcl
resource "aws_iam_policy" "authguard" {...}
resource "aws_iam_role" "authguard" {...}
resource "aws_iam_role_policy_attachment" "authguard" {...}
resource "aws_lambda_function" "authguard" {...}
resource "aws_lambda_permission" "invoke_permission_from_cognito" {
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.authguard.function_name
  principal     = "cognito-idp.amazonaws.com"
  source_arn    = aws_cognito_user_pool.userpool.arn
}
```

### Lambda (failcount)

CloudWatch Logs に以下のような Lambda をサブスクさせてログから失敗回数のカウントに繋げます。

```javascript
const ddb = new DynamoDBClient({});
const tableName = "lockout-ddb";

const gunzip = (buf) =>
  new Promise((resolve, reject) =>
  zlib.gunzip(buf, (err, out) => (err ? reject(err) : resolve(out))));

exports.handler = aysnc (event) => {
  const compressed = Buffer.from(event.awslogs.data, "base64");
  const decompressed = await gunzip(compressed);
  const payload = JSON.parse(decoder.decode(decompressed));

  for (const le of payload.logEvents ?? []) {
    const parsed = JSON.parse(le.message);
    const msg = parsed?.message ?? {};
    const userSub = msg.userSub;
    const lastTrial = msg.challenges.at(-1);

    const isPasswordTrial = [
      "Password:Success", 
      "Password:Failure"
    ].includes(lastTrial);
    const isTrialSuccess = lastTrial === "Password:Success";

    if (!userSub || !usPasswordTrial) {
      continue;
    }

    const now = Math.floor(Date.now() / 1000);

    if (isTrialSuccess) {
      await ddb.send(
        new UpdateItemCommand({
          TableName: tableName,
          Key: { userSub: { S: userSub }},
          UpdateExpression: "SET failCount = :zero",
          ExpressionAttributeValues: {
            ":zero": { N: "0" },
          },
        })
      );
    } else {
      const res = await ddb.send(
        new UpdateItemCommand({
          TableName: tableName,
          Key: { userSub: { S: userSub }},
          UpdateExpression: "ADD failCount :inc",
          ExpressionAttributeValues: {
            ":inc": { N: "1" },
          },
          ReturnValues: "UPDATED_NEW",
        })
      );

      const failCount = Number(res.Attributes?.failCount?.N || 0);
      if (failCount >= MAX_LOGIN_ATTEMPTS) {
        await ddb.send(
          new UpdateItemCommand({
            TableName: tableName,
            Key: { userSub: { S: userSub }},
            UpdateExpression: "SET lockUntil = :lu"
            ExpressionAttributeValues: {
              ":lu": { N: String(now + LOCKOUT_SECONDS) },
            }
          })
        );
      }
    }

    await ddb.send(
      new UpdateItemCommand({
        TableName: tableName,
        Key: { userSub: { S: userSub }},
        UpdateExpression: "SET hasQueue = :flag",
        ExpressionAttributeValues: { ":flag": { BOOL: false }}
      })
    );
  }
}
```

terraform 周りはこれだけ設計すれば良いと思います。
SignIn のイベントだけ処理する感じにしています。

```hcl
resource "aws_iam_policy" "failcount" {...}
resource "aws_iam_role" "failcount" {...}
resource "aws_iam_role_policy_attachment" "failcount" {...}
resource "aws_lambda_function" "failcount" {...}
resource "aws_cloudwatch_log_subscription_filter" "signin" {
  log_group_name  = aws_cloudwatch_log_group.cognito.name
  destination_arn = aws_lambda_function.failcount.arn
  filter_pattern  = "{ ($.eventSource = \"USER_AUTH_EVENTS\") && ($.message.eventType = \"SignIn\") }"
}
resource "aws_lambda_permission" "invoke_permission_from_cloudwatch" {
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.failcount.function_name
  principal     = "logs.<region>.amazonaws.com"
  source_arn    = "${aws_cloudwatch_log_group.userlog.arn}:*"
}
```

### おわりに
Hosted UI の形を保ったままロックアウトをカスタム制御する機構について述べてきましたが、ネイティブ実装範囲外の制御を入れると結構面倒っすね。
ロックアウト機構のカスタマイズがネイティブ実装で可能になると嬉しくなる人結構いると思うんですが、あんまり需要がないんでしょうかね？

今回実装しませんでしたが、パスワードの有効期限などの仕組みに関しては CloudTrail で `ChangePassword` 等のイベントを EventBridge 通して Lambda にトリガすれば作れそうな気がします。こっちの方がロック機構考えなくて良い分実装難易度は簡単でしょうね（本当に？誰か記事出してくれるとありがたいです）。
`expiredAt` のような key を DynamoDB に追加して一緒に管理することになるのかなあ。
