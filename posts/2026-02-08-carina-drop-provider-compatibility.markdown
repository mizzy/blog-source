---
title: Carina aws providerとawscc providerに互換性を持たせるのをやめた
date: 2026-02-08 12:34:04 +0900
---

以前、[Carina aws providerとawscc provider](https://mizzy.org/blog/2026/01/29/2/)という記事で、aws providerとawscc providerのスキーマをCloud Control APIベースで統一し、`aws`と`awscc`を置換するだけで切り替えられるようにしている、と書いた。この方針をやめた。

---

## やめた理由

aws providerはSDKを直接叩くので、SDKのAPI構造に合わせた方が自然だし、パフォーマンスも有利になる。無理にawscc providerと同じスキーマに合わせると、SDKの良さを活かしきれない。

一方、awscc providerはCloud Control APIを使っているので、CloudFormationのスキーマにそのまま合わせるのが最も自然。両者の性質が異なるのに、無理に互換性を持たせようとすると後々無理が出てきそうだな、と思った。

また、仮にプロバイダ間で切り替えたい場合でも、今の時代なら生成AIに変換を任せれば良い。人間が手作業で置換することを前提にスキーマの互換性を維持するより、それぞれのプロバイダに最適な設計にした方が合理的だろう。

---

## 変更内容

### awscc providerの命名規則変更

awscc providerのリソース名を、CloudFormationのリソースタイプ名に完全に合わせるようにした。

変更前:

```crn
awscc.vpc {
  name       = "example-vpc"
  cidr_block = "10.0.0.0/16"
}
```

変更後:

```crn
awscc.ec2_vpc {
  name       = "example-vpc"
  cidr_block = "10.0.0.0/16"
}
```

CloudFormationでは`AWS::EC2::VPC`というリソースタイプ名なので、awscc providerでも`awscc.ec2_vpc`とする。以前は`awscc.vpc`のようにサービス名を省略していたが、CloudFormationのスキーマから自動生成する際にも、この方が素直にマッピングできる。

### aws providerのスキーマ

aws providerのスキーマは引き続きCloudFormationのスキーマをベースにするが、awscc providerとの完全な互換性は維持しない。SDKのAPI構造に合わせた方が自然な場合は、そちらに寄せる方針にした。具体的にどう変えるかはまだこれからだが、大きな方針として決めた。

---

## まとめ

以前は「`aws`と`awscc`を置換するだけで切り替えられる」ことを目指していたが、両者の性質が根本的に異なるため、それぞれに適したスキーマ設計をした方がよい、という結論になった。awscc providerはCloudFormationに忠実に、aws providerはSDKに寄せる。ユーザーにとっても、それぞれのプロバイダの特性が明確になる方がわかりやすいはず。
