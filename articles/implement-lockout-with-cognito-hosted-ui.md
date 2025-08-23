---
title: "Cognito Hosted UI 上でロックアウトをカスタム制御する"
emoji: "💥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Cognito"]
published: false
---

## はじめに
社内向けの小規模アプリケーションを作成する際に、Cognito の Hosted UI を利用して簡易的に認証基盤を構築することは多いはず。
Hosted UI を利用すれば自前で Cognito の API を叩くカスタムページを作成せずに済むので確かに便利ですよね。

ただ、実際の案件上では企業のパスワードポリシーが存在して、これに準拠するように制御をカスタマイズしなければならないこともあるはず。
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
ただログの配送までにラグがあってめんどくさいので flag を立てて処理中は一時的にロックアウトさせる。

- 必要コスト
  - Cognito Advanced Security Function (ASF) **PLUS**
  - DynamoDB, CloudWatch Logs の使用料金
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
    - 一時ロックアウトは数秒から数十秒にわたることがしばしばある。こればっかりはどうしようもない。
  - パスワード成功かつ MFA 失敗時、失敗カウントは 0 になる。
    - MFA 失敗にはロックアウト、複数回の失敗を含む
  - パスワード成功かつ MFA 成功時、失敗カウントは 0 になりユーザはリソースにアクセスする
    - MFA 失敗後からの成功を含む
- 要注意点
  - パスワード成功が連続する時も、一時ロックアウトが発生する。

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

- Password 通過・失敗時点で `$.message.challenges[]` に `Password:Success` / `Password:Failure` が追加されて発行
- MFA 通過・失敗時点で `Mfa:Success` / `Mfa:Failure` が追加されて発行、失敗したら失敗した回数だけ追加されて発行
- MFA タイムアウト時は発行されない

Password 認証前に Pre Authentiation Lambda が起動することを踏まえ、結局 Password 通過・失敗時点のログだけ拾ってやればよいことになるので、`$.message.challenges[]`の末尾に `Password:Success` / `Password:Failure` が来たらカウンタを更新すれば良い。`Mfa:*` 系が来た場合は放棄で OK.

## 実装

### DynamoDB

失敗カウンタと後述するロック機構を実現するために DB が必要。
こういうのは強整合性適用した DynamoDB に任せれば良いという SAA のお告げがあったので DynamoDB を選択したが、正直社内アプリレベルなら Cognito User pool の attribute に書き込んでも良い気がする。
ただコスト面でスケールしないのでユーザ数が大きい場合は注意。

```hcl
resource "aws_dynamodb" "lockout" {
    ""
}
```

今回のシステムは各パスワード認証試行時に一時的な（数秒から数十秒）ロックアウトが毎回発生するというデメリットがある。
この一時ロックアウトは Cognito から CloudWatch Logs に送出される際に少々ラグが発生していることに起因しており、カウンタに失敗回数が反映されるまで試行させたくないという背景がある。
もしこのロック機構がない場合、カウント反映前に試行ができてしまうので n 回の失敗で t 時間ロックアウトしたいという要件を確実に守れなくなる。
もしこの一時ロックアウトを消したいのであれば Hosted UI を使わないという選択肢になるので、工数と相談して下さいねー。

### Cognito

Cognito user pool は大体こんな感じになる。

```hcl
resource "aws_cognito_user_pool" "this" {
    "plan" = "PLUS"
    "mode" = "AUDIT"
    lambda {
        pre_authentication = aws_lambda.authguard.arn
    }
}
```

Plus プランは監査のみモードでログが出ます。フル機能にする必要ないです。

### Lambda (authguard)

ロックアウトを実現する Cognito Pre Authentication の Lambda を書くとしたらこんな感じ。

```javascript
const test = 0;
```

terraform 周りはこれだけ設定すれば良いと思います。

```hcl
test = test
```

### Lambda (failcount)

失敗カウントを実現する CloudWatch Logs に subscripted した Lambda を書くならこんな感じです。

```javascript
const test = 0;
```

terraform 周りはこれだけ設計すれば良いと思います。

```hcl
test = test
```