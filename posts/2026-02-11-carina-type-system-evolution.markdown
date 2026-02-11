---
title: Carinaの型システム強化
date: 2026-02-11 20:44:44 +0900
---

[前回のawscc providerの自動スキーマ生成の記事](https://mizzy.org/blog/2026/02/04/1/)では、CloudFormationスキーマからRustコードを自動生成するとともに、元がString型であっても、descriptionから列挙型であることが推測される場合には、名前空間付き列挙型に変換する仕組みを導入した。

今回も型まわりの強化を進めてみた。

---

## リソースID型の厳密化

以前は、VPC IDもSubnet IDもSecurity Group IDも、すべて`String`型として扱っていた。その後`AwsResourceId`型を導入したが、それだとVPC IDを期待する属性にSubnet IDを渡しても型エラーにならない。

そこで、リソースタイプごとにプレフィックスバリデーション付きの専用型を導入した。

```rust
pub fn vpc_id() -> AttributeType {
    AttributeType::Custom {
        name: "VpcId".to_string(),
        base: Box::new(AttributeType::String),
        validate: |value| {
            if let Value::String(s) = value {
                validate_prefixed_resource_id(s, "vpc")
            } else {
                Err("Expected string".to_string())
            }
        },
        namespace: None,
    }
}
```

現時点で以下の12種類の専用型がある。

| 型 | プレフィックス | 例 |
|---|---|---|
| VpcId | `vpc-` | `vpc-1a2b3c4d` |
| SubnetId | `subnet-` | `subnet-0123456789abcdef0` |
| SecurityGroupId | `sg-` | `sg-12345678` |
| InternetGatewayId | `igw-` | `igw-12345678` |
| RouteTableId | `rtb-` | `rtb-abcdef12` |
| NatGatewayId | `nat-` | `nat-0123456789abcdef0` |
| VpcPeeringConnectionId | `pcx-` | `pcx-12345678` |
| TransitGatewayId | `tgw-` | `tgw-0123456789abcdef0` |
| VpnGatewayId | `vgw-` | `vgw-12345678` |
| EgressOnlyInternetGatewayId | `eigw-` | `eigw-12345678` |
| VpcEndpointId | `vpce-` | `vpce-0123456789abcdef0` |
| IpamPoolId | `ipam-pool-` | `ipam-pool-0123456789abcdef0` |

---

## IamPolicyDocument型

IAMロールの信頼ポリシーやインラインポリシーなど、IAMポリシードキュメントはネストされたリストやマップを含むため、`Map<String>`型では表現できない。そこで`IamPolicyDocument`型を導入し、構造レベルのバリデーションを追加した。

```rust
pub fn iam_policy_document() -> AttributeType {
    AttributeType::Custom {
        name: "IamPolicyDocument".to_string(),
        base: Box::new(AttributeType::Map(Box::new(AttributeType::String))),
        validate: |value| validate_iam_policy_document(value),
        namespace: None,
    }
}
```

バリデーターは以下をチェックする。

- `version`が`"2012-10-17"`か`"2008-10-17"`であること
- `statement`がリストであること
- 各Statementの`effect`が`"Allow"`か`"Deny"`であること

現時点ではまだ基本的な構造チェックのみなので、`action`や`resource`、`principal`のバリデーションなど、今後さらに厳密にしていく予定。

`.crn`ファイルではこんな感じで使う。

```crn
let flow_log_role = awscc.iam_role {
  name      = "flow-log-role"
  role_name = "flow-log-role"
  assume_role_policy_document = {
    version = "2012-10-17"
    statement = [
      {
        effect    = "Allow"
        principal = { service = "vpc-flow-logs.amazonaws.com" }
        action    = "sts:AssumeRole"
      }
    ]
  }
}
```

Terraformでは、`aws_iam_policy_document` data sourceを使えば、補完や構文チェックができるが、inline policyの場合はただのJSON文字列なので、補完も構文チェックもできない。Carinaでは`IamPolicyDocument`型を使うことで、インラインポリシーの記述でも補完と構文チェックを可能にしたいと考えている。

---

## リソース間の型チェック

リソース参照時の型の不一致も検出できるようにした。

たとえば、こんなコードを書いたとする。

```crn
let vpc = awscc.ec2_vpc {
  name       = "my-vpc"
  cidr_block = "10.0.0.0/16"
}

awscc.ec2_vpc {
  name              = "another-vpc"
  ipv4_ipam_pool_id = vpc.vpc_id  # これは間違い！
}
```

`ipv4_ipam_pool_id`は`IpamPoolId`型を期待しているが、`vpc.vpc_id`は`VpcId`型。以前はどちらも`String`だったので素通りしていたが、今はバリデーション時に型の不一致としてエラーになる。

型の互換性ルールはシンプルにしている。

- 同じ型 → OK
- どちらかが`String`→ OK（後方互換のため）
- 異なるCustom型 → エラー

このチェックはCLIの`validate`/`plan`/`apply`コマンドで実行されるほか、LSPでもリアルタイムにエディタ上で警告が表示される。
