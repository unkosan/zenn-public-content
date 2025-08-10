---
title: "Terragrunt の generate block を応用して removed block の for_each を実現する"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terragrunt", "terraform"]
published: false
---

## 注意
この記事は terragrunt の利用を前提としています。
生の Terraform を利用している方は "removed / moved block の基本" まで確認後、スクリプトを作成するか手で作成する感じになると思います。
（博識な方いらっしゃればこれに相当する機能教えてくださると助かります）

## はじめに
Terragrunt (Terraform) を利用していると、既存リソースを destroy せず tfstate からのみ削除したい場面があると思います。
筆者が実際に経験した例で言えば、version の更新により破壊的変更が入ったリソースを一度管理外に置かせて、後に import するという文脈で必要になったことがありました。
その際に便利なのが `removed` ブロックです。

通常、この要件を達成するには `terragurnt state rm` コマンドを実行すると思いますが、`removed` ブロックを作成することにより terragrunt apply を通した削除が可能になります。
すなわち、terragrunt apply の CI/CD さえ構築されていれば、余計な shell script を組まず自動的にリソース削除を実施することができます。

しかし、`removed` ブロックには大きな制限があります。それは **`for_each` や `count` に対応していない**ことです。
`terragrunt state rm` を含んだスクリプトの作成と実行を CI/CD 範囲外で行わなければならなくなるため、DevOps が成熟して商用環境が基本的に CI/CD からしかアクセスできない状況だったら割と面倒に感じると思います。

本記事では、この制限を Terragrunt の `generate block` 機能を使って回避し、動的に `removed block` を構築して terragrunt apply で削除する方法を紹介します。

## Terraform における removed / moved block の基本

Terraform の `removed` ブロックは、リソース定義の削除を宣言的に行うための仕組みです。

```hcl
# resource "aws_instance" "to_be_deleted" {
#   ...
# }

removed {
  from = aws_instance.to_be_deleted
  lifecycle {
    destroy = false
  }
}
```

通常、リソース定義を tf ファイルから削除すると destroy が走りますが、`removed` ブロックを記載することによって、実際のリソースの削除は行われず tfstate からのリソース定義の削除だけが行われるようになります。
実際に plan, apply の実行時には、 `... will be destroyed` から `... will not be destroyed` への表示へ変化します。

しかし、`removed` ブロックは `for_each` や `count` 属性を持ちません。それだけではなく以下のような配列の指定も受け付けないため、単純に `for_each` の各アイテムを列挙するという力技もそのままでは通用しません。

```hcl
removed {
  from = aws_instance.to_be_deleted["some_key"]
  lifecycle {
    destroy = false
  }
}
```

これを解決するために、追加で `moved` ブロックが必要になります。
`moved` ブロックはリソースの名前を宣言的に変更する terraform の機能です。

```hcl
# resource "aws_instance" "before_moved" {
#   for_each = ...
#   ...
# }
resource "aws_instance" "after_moved" {
  ...
}

moved {
  from = aws_instance.before_moved["some_key"]
  to   = aws_instance.after_moved_some_key
}
```
これによりリソース定義のインスタンスの名称が変更されます。
`before_moved` と `after_moved` の属性値が異なる場合、move と同時に属性も変更されるので、差分がないか plan で念入りに確認するのをお願いしたいです。

この二つの block を terraform apply することで `for_each` や `count` を削除することを目指します。 terragrunt を利用されていない読者の方は、ここまでの情報を元にご自身で tf ファイルをご記載ください。

公式情報も貼っておきますので、詳しく知りたい方はこちらからご参照ください。

https://github.com/hashicorp/terraform/issues/34439

https://support.hashicorp.com/hc/en-us/articles/34185721057555-Removing-a-Resource-from-the-Terraform-State-Using-the-removed-block

## Terragrunt の generate block の基本

本記事では上述した `moved`, `removed` block を `generate` ブロックで生成することで `for_each` を実装します。その前に `generate` ブロックに関して軽くおさらいします。

Terragrunt の `generate` ブロックは、`terragrunt.hcl` 実行時に動的にファイルを生成する機能です。
これは terragrunt 特有の機能であり、`terragrunt.hcl` もしくはそれに include される `.hcl` ファイルに記載することで実行可能です。
動的に作成されたファイルは terragrunt plan, apply を実行すると作成されて読み込まれます。

例えば、provider 設定をトップ階層の `.hcl` ファイルに `generate` ブロックとして記載し、商用環境、ステージ環境、開発環境の各モジュール単位の `terragrunt.hcl` で include することにより、コードを DRY に保ちながらも環境による変動を動的に適用することができますね。以下のような感じです

**ディレクトリ構成**
- `infra/`
  - `root.hcl`
  - `dev/`
    - `terragrunt.hcl`
  - `stg/`
    - `terragrunt.hcl`
  - `prd/`
    - `terragrunt.hcl`

**`root.hcl`**
```hcl
locals {
  env_regions = {
    dev = "ap-northeast-1"
    stg = "ap-northeast-1"
    prd = "us-east-1"
  }
  env = basename(get_terragrunt_dir())
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<-EOF
    provider "aws" {
      region = "${local.env_regions[local.env]}"
    }
  EOF
}
```
このような構成にすることにより、provider の定義は `root.hcl` に集約されたまま環境差分を適用することができます。

## Generate block を利用した for_each の代替

今回はこの generate block の中で `for_each` を実装します。
`aws_instances.examples["..."]` を tfstate から削除したい場合、最終的には以下のようになるはずです。

move 実行スクリプト
```hcl
generate "moved_resources" {
  path      = "moved_resources"
  if_exists = "overwrite"
  contents  = <<-EOF
  %{ for elem in local.parameters ~ }
    resource "aws_instances" "examples_${elem}" {
      ...
    }
    moved {
      from = aws_instances.examples["${elem}"]
      to   = aws_instances.examples_${elem}
    }
  %{ endfor ~ }
  EOF
}
```

state rm 実行スクリプト
```hcl
generate "removed_resources" {
  path      = "removed_resources"
  if_exists = "overwrite"
  contents  = <<-EOF
  %{ for elem in local.parameters ~ }
    removed {
      from = aws_instances.examples_${elem}
      lifecycle = {
        destroy = false
      }
    }
  %{ endfor ~ }
  EOF
}
```
CI/CD を走らせる際は、それぞれの generate block を別の PR の `.hcl` ファイルに作成して順にマージする形になると思います。これで `for_each` が事実上可能になります。お疲れ様でした。

## おわりに

自分の環境下だけか不明ですが、generate block を編集したり消したりすると、`.terragrunt-cache` に編集前ファイルが残ってしまいローカルでうまく作動しないことがあります。
CI/CD で実行される場合は関係ないですが、望ましくない挙動をしていたら削除してみるのも一つの手ではあるかなと思います。

terraform も terragrunt も積極的に開発が進められている段階（特に terragrunt はプレリリースの状況）なので、今後の発展に期待していきたいですね。

## 参考文献
https://github.com/hashicorp/terraform/issues/34439