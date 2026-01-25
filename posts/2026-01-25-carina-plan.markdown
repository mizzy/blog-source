---
title: carina planで依存関係を可視化
date: 2026-01-25 15:14:36 +0900
---

[Carina](https://github.com/mizzy/carina)の`plan`コマンドで、リソース間の依存関係をツリー構造で表示できるようにした。

![carina plan の出力](/images/2026/01/carina-plan.png)

Terraformの`plan`コマンドは、作成・変更・削除されるリソースがフラットなリストで表示される。どのリソースが作成されるかはわかるが、リソース間の依存関係は出力からは読み取れない。

Carinaの`plan`では、依存関係に基づいてリソースがツリー構造で表示される。上の例だと、VPCの下にSecurity Groupがあり、その下にIngress Ruleがある、という階層構造が一目でわかる。

これにより、planの出力を見るだけで、リソース間の依存関係を把握できる。
