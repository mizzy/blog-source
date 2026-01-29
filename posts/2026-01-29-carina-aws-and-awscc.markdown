---
title: Carina aws providerとawscc provider
date: 2026-01-29 22:07:30 +0900
---

[Carina](https://github.com/mizzy/carina)に、aws providerに加えてawscc providerを実装してみた。

---

## 背景

以前、[AWS Cloud Control APIは遅い？](https://mizzy.org/blog/2026/01/24/2/)という記事で、Cloud Control APIは遅いので採用をやめた、と書いた。しかし、その後考えが変わった。

Cloud Control APIには以下のメリットがある。

- CloudFormationと同じリソースモデルを使っているため、リソースのスキーマが公式に定義されている
- 新しいリソースタイプやプロパティが追加された場合、CloudFormationのスキーマが更新されれば自動的にサポートできる
- 統一されたAPIで様々なリソースを操作できる

確かに遅いが、遅さが許容できるユースケースもあるはずだし、もっと多くのリソースを扱ってみると、実はCloud Control APIの方が便利、という場面も出てくるかもしれない。

また、両方のプロバイダを用意しておけば、ユーザーがユースケースに応じて選択できる。

というわけで、aws provider（直接SDKを使う高速な方式）とawscc provider（Cloud Control APIを使う方式）の両方を実装しはじめた。

---

## DSL構文

### awsプロバイダ

```crn
provider aws {
  region = aws.Region.ap_northeast_1
}

aws.vpc {
  name                 = "example-vpc"
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  instance_tenancy     = aws.vpc.InstanceTenancy.dedicated
}
```

### awsccプロバイダ

```crn
provider awscc {
  region = aws.Region.ap_northeast_1
}

awscc.vpc {
  name                 = "cloudcontrol-vpc"
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  instance_tenancy     = awscc.vpc.InstanceTenancy.dedicated
}
```

基本的な構文は同じだが、以下の点が異なる。

- プロバイダ名が`aws`と`awscc`
- リソースタイプが`aws.vpc`と`awscc.vpc`
- enum値の名前空間が`aws.vpc.InstanceTenancy`と`awscc.vpc.InstanceTenancy`

`aws`と`awscc`を置換するだけで切り替えられるようになっている。今後追加するリソースタイプも、基本的には両方とも同じスキーマで定義できるようにして、簡単に切り替えられるようにする予定。そうすることで、どちらがどういうユースケースに適しているかを試しやすくなるはず。
