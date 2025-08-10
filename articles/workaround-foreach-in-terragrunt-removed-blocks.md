---
title: "Terragrunt の generate block を応用して removed block の for_each を実現する"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terragrunt", "terraform"]
published: false
---

## 注意
この記事は terragrunt の利用を前提としています。
生の Terraform を利用している方は頑張って shell script を組むか一リソースずつ手で作成してください。
（博識な方いらっしゃればこれに相当する機能教えてくださると助かります）

## はじめに
Terragrunt (Terraform) を利用していると、既存リソースの削除を安全に行いたい場面があります。
筆者が実際に経験した例で言えば、version の更新により破壊的変更が入ったリソースを tfstate からの削除するのに必要となることがありました。
その際に便利なのが `removed` ブロックです。

通常、この要件を達成するには `terragurnt state rm` コマンドを実行すると思いますが、`removed` ブロックを作成することにより terragrunt apply を通した削除が可能になります。
すなわち、terragrunt apply の CICD さえ構築されていれば、余計なスクリプトを組まず自動的にリソース削除を実施することができます。

しかし、`removed` ブロックには大きな制限があります。それは `for_each` や `count` に対応していないことです。

破壊的変更を跨いだマイグレーション等では、`for_each` を含んだリソース定義を全部削除することもあるため、これができないとなると `terragrunt state rm` を含んだスクリプトの作成と実行を CICD 範囲外で行わなければならなくなると思います。商用環境が CD からしかアクセスできない状況だったら尚更面倒くさいですね。

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
`before_moved` と `after_moved` の属性値が異なる場合、move と同時に属性も変更されるので、差分がないか plan で念入りに確認することをお願いします。

この二つの block を terraform apply することで `for_each` や `count` を削除することを目指します。 terragrunt を利用されていない読者の方は、ここまでの情報を元にご自身で tf ファイルをご記載ください。

公式情報も貼っておきますので、詳しく知りたい方はこちらからご参照ください。

https://github.com/hashicorp/terraform/issues/34439

https://support.hashicorp.com/hc/en-us/articles/34185721057555-Removing-a-Resource-from-the-Terraform-State-Using-the-removed-block

## 参考文献
https://github.com/hashicorp/terraform/issues/34439