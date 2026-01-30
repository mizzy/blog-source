---
title: Carinaで列挙型値の省略記法をサポート
date: 2026-01-30 16:45:00 +0900
---

[Carina](https://github.com/mizzy/carina)で、列挙型の値を省略して書けるようにした。

## 以前の書き方

これまでは、VPCのインスタンステナンシーを指定する場合、完全な名前空間付きの形式で書く必要があった。

```crn
awscc.vpc.vpc my_vpc {
    cidr_block       = "10.0.0.0/16"
    instance_tenancy = awscc.vpc.InstanceTenancy.dedicated
}
```

`awscc.vpc.InstanceTenancy.dedicated`という記述は正確だが、冗長に感じられる。書くときは補完機能を使うので問題ないが、読みにくい。

## 新しい書き方

今回の変更で、3つの形式をサポートした。

```crn
// 完全形式（従来通り）
instance_tenancy = awscc.vpc.InstanceTenancy.dedicated

// 型名 + 値
instance_tenancy = InstanceTenancy.dedicated

// 値のみ
instance_tenancy = dedicated
```

属性のスキーマには、どの名前空間に属するかの情報がある。`instance_tenancy`属性は`awscc.vpc`名前空間に属し、型名は`InstanceTenancy`だとわかっている。この情報を使って、省略形式を完全形式に展開する。

## 実装

パーサーは、未知の識別子を見つけると`UnresolvedIdent`として保持する。スキーマの検証時に、属性の名前空間情報を使って完全な形式に展開する。

LSPの補完も更新し、省略形式で候補を表示するようにした。`dedicated`や`default`といった短い形式で補完される。

## 関連リンク

- [Add shorthand enum value support with schema-based namespace resolution by mizzy · Pull Request #34 · mizzy/carina](https://github.com/mizzy/carina/pull/34)
