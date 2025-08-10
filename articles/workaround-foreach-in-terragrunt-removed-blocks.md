---
title: "Terragrunt の generate block を応用して removed block の for_each を実現する"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
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
