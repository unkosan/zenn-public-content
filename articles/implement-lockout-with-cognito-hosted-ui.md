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

## 現状確認

調査したところ、CloudTrail から成功・失敗の証跡を取得することはできるらしいとのこと、以下を参照。

https://dev.classmethod.jp/articles/how-to-check-the-cognito-authentication-log/

https://qiita.com/shinichi_yoshioka2/items/c1af96f83502a92c5998

これを EventBridge と連携して Lambda で dynamodb に書き込みするか？と仮説を立てていたところ `InitiateAuth` はどうやら Hosted UI からの認証イベントでは発行できないことが発覚。この方針は独自 Web UI を構築しカスタムロジックで API を叩く際の話に限られることになった。

- 確認済みの前提情報
  - Password 認証前に Pre Authentication Lambda が起動
  - Password 通過・失敗時点で `Password:Success` / `Password:Failure` が発行
  - MFA 通過・失敗時点で `Mfa:Success` / `Mfa:Failure` が発行
    - MFA タイムアウトでは `Mfa:Failure` が発行されない

- 正直社内アプリレベルなら userpool の attribute に書き込んでも良い気はするが、コスト面でスケールしないので注意。