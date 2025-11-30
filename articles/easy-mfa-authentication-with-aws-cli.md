---
title: "Access-key based でも AWS CLI の MFA 認証を簡便化する"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "iam"]
published: false
---

## 背景

先日 (2025/11) のアップデートで aws cli に login コマンドが実装され、煩雑だった MFA 認証もマネジメントコンソール経由で簡単にできるようになりましたね。

https://aws.amazon.com/jp/blogs/security/simplified-developer-access-to-aws-with-aws-login/

運用も楽そうなので自分も基本的にこれを使うの概ね賛成ですが、要件によってはブラウザ依存のため攻撃表面が増大してセキュリティ的に不安だとか、ネットワークやハードウェアの要因でそもそもブラウザを開くことができない^[[実は URL コピペさえできれば remote オプションで aws login を別マシン上で認証可能](https://dev.classmethod.jp/articles/run-aws-login-from-remote-host/)]環境にいたりする方もいるのではないかと思います。
アクセスキー・シークレットキーを利用する旧来の方法に関しても、しばらくはまだ一定のニーズはあるように見えていますね。

そんな方々に朗報ですが、AWS CLI v2.32.3 から `aws configure mfa-login` サブコマンドが実装されました。これを使うことで公式 CLI だけでも MFA が楽に認証できるようになっています。
本記事では、`mfa-login` を使って簡単に MFA 認証を行う方法を紹介します。

## 旧来の MFA 認証方法

今までの認証方法はこんな感じでした。概ね以下のような手順を踏んでいた人が多いのではないでしょうか。
マウスを使ってコマンド間を行ったり来たり、疲れますよね。

```shell
# 長期アクセストークンを登録
$ aws configure --profile auth
AWS Access Key ID [None]: <access_key>
AWS Secret Access Key [None]: <secret_access_key>
Default region name [None]: ap-northeast-1
Default output format [None]: json

# アカウント ID 等を含む ARN を指定して、MFA トークンとともに指定
$ aws sts get-session-token --profile auth --serial-number arn:aws:iam::<account_id>:mfa/<user_name> --token-code <mfa_token>
{
    "Credentials": {
        "AccessKeyId": <mfa_access_key>,
        "SecretAccessKey": <mfa_secret_access_key>,
        "SessionToken": <mfa_session_token>,
        "Expiration": "2025-11-28T18:16:07+00:00"
    }
}

# アクセスキー、シークレットキーをわざわざ一つずつコピペして MFA 用のプロファイルを作成
$ aws configure
AWS Access Key ID [None]: <mfa_access_key>
AWS Secret Access Key [None]: <mfa_secret_access_key>
Default region name [None]: ap-northeast-1
Default output format [None]: json

# 一時トークンもコピペしてセッションに追加
$ aws configure set aws_session_token "<mfa_session_token>"
```

この操作が煩わしくてユーザ側でスクリプトを作成^[[プロファイルまで作成するスクリプト](https://dev.classmethod.jp/articles/aws-cli-mfa/)]^[[環境変数として各一時トークンを設定するスクリプト](https://qiita.com/hoto17296/items/38f8575265a514660745)]したり、サードパーティの [aws-vault](https://github.com/99designs/aws-vault) を導入したことも人によってはあるかもしれないですね。

## mfa-login を利用した認証

version 2.32.3 以上から利用可能になる `mfa-login` サブコマンドでは、上記の煩雑な MFA 認証を一行にまとめてくれます。公式ドキュメントは以下です。
https://docs.aws.amazon.com/cli/latest/reference/configure/mfa-login.html

AWS CLI の version が v2.32.3 以上であることを確認しましょう。
バージョンが古い場合は、各環境に応じてアップデートしておきましょう。

```shell
$ aws --version
```

アクセスキーとシークレットキーを登録します。
default のプロファイルに書くと後の工程で上書きされるため、長期で保持するキーは別のプロファイルに保管しておくのが良いかと思います。今回の場合は `auth` としておきます。

```shell
$ aws configure --profile auth
# 省略
```

`auth` のプロファイルに MFA device の ARN を指定する `mfa_serial` の値を付与します。
`mfa-login` は `--serial-number` オプションを省略した場合、このプロファイル値を参照してくれます。

```
$ aws configure set mfa_serial "arn:aws:iam::<accound_id>:mfa/<user_name>" --profile auth
```

ここまでで初期設定は完了です。

利用時は以下のコマンドを実行して認証情報を取得・保管します。
対話的に MFA のトークンコードが聞かれるので `<mfa_token>` に 6 桁のコードを入れてあげてください。

```
$ aws configure mfa-login --profile auth --update-profile default

MFA token code: <mfa_token>
Temporary credentials written to profile 'default'
Credentials will expire at 2025-11-28T20:41:05+00:00
To use these credentials, specify --profile default when running AWS CLI commands
```

`profile` が sts で利用する長期アクセストークン、`update-profile` が MFA 認証後の一時トークンとアクセスキー、シークレットキーが入れられるプロファイルです。既定だと `<session_id>-<mfa_device_name>` としてプロファイル名がつけられ使いにくいので `default` を指定しています。名前は任意ですが、`default` 以外だと毎回 aws cli 使用時にプロファイル名を追加しなければならなくなることには注意してください。

## 蛇足

MFA の ARN を省略する機能、実は 7 年くらい放置されていました。
結構目につく欠点だと思うのですが、対応にここまで時間がかかったのは不思議すね。
https://github.com/aws/aws-cli/issues/9019

あとは update-profile のデフォルト名、何とかならんすかね。