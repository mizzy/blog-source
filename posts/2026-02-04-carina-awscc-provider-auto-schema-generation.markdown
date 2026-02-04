---
title: Carina awscc providerの自動スキーマ生成
date: 2026-02-04 09:40:42 +0900
---

[Carina](https://github.com/mizzy/carina)を1週間ほどコードを見ずにClaude Codeにつくらせて、それっぽく動くようにはなったけど、そろそろコードを読んで指示出さないといけないフェーズに入ったかな、と思い、まずはawscc providerのコードを読んでいる。

実際にコードを読んでみると、awscc providerはAWS Cloud Control APを利用しているので、aws-sdk-cloudcontrol crateだけを利用すればいいはずなのに、なぜかaws-sdk-ec2 crateも利用している、といったおかしなことになっていた。

また、プロバイダー用スキーマは、[CloudFormation resource provider schemas](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/resource-type-schemas.html)から自動生成できるはずだがそうなっておらず、Claude Codeが手動？でスキーマを定義していた。（これはそのように指示を出していない自分が悪いのだが。）

というわけで自動生成するようにしてもらった。こんな感じで、スキーマをRustコードの形で自動生成してくれる。

```rust
//! internet_gateway schema definition for AWS Cloud Control
//!
//! Auto-generated from CloudFormation schema: AWS::EC2::InternetGateway
//!
//! DO NOT EDIT MANUALLY - regenerate with carina-codegen

use super::tags_type;
use carina_core::schema::{AttributeSchema, AttributeType, ResourceSchema};

/// Returns the schema for ec2_internet_gateway (AWS::EC2::InternetGateway)
pub fn ec2_internet_gateway_schema() -> ResourceSchema {
    ResourceSchema::new("awscc.ec2_internet_gateway")
        .with_description("Allocates an internet gateway for use with a VPC. After creating the Internet gateway, you then attach it to a VPC.")
        .attribute(
            AttributeSchema::new("internet_gateway_id", AttributeType::String)
                .with_description(" (read-only)"),
        )
        .attribute(
            AttributeSchema::new("tags", tags_type())
                .with_description("Any tags to assign to the internet gateway."),
        )
}
```

VPCの`cidr_block`は、CloudFormation resource provider schemaでは`string`型になっている。

```json
        "CidrBlock": {
            "description": "The IPv4 network range for the VPC, in CIDR notation. For example, ``10.0.0.0/16``. We modify the specified CIDR block to its canonical form; for example, if you specify ``100.68.0.18/18``, we modify it to ``100.68.0.0/18``.\n You must specify either``CidrBlock`` or ``Ipv4IpamPoolId``.",
            "type": "string"
        },
```

だが、Carinaは強い型付けを目指しているので、ドメイン固有型である`cidr`型に変換している。これは指示しなくてもClaude Codeが勝手にそうしてくれた。

```rust
        .attribute(
            AttributeSchema::new("cidr_block", types::cidr())
                .with_description("The IPv4 network range for the VPC, in CIDR notation. For example, ``10.0.0.0/16``. We modify the specified CIDR block to its canonical form; for exam..."),
        )
```

VPCの`instance_tenancy`属性は、CloudFormation resource provider schemaでは`string`型になっているが、取り得る値は3つに限定されていることがdescriptionから読み取れる。

```json
        "InstanceTenancy": {
"description": "The allowed tenancy of instances launched into the VPC.\n  +  ``default``: An instance launched into the VPC runs on shared hardware by default, unless you explicitly specify a different tenancy during instance launch.\n  +  ``dedicated``: An instance launched into the VPC runs on dedicated hardware by default, unless you explicitly specify a tenancy of ``host`` during instance launch. You cannot specify a tenancy of ``default`` during instance launch.\n  \n Updating ``InstanceTenancy`` requires no replacement only if you are updating its value from ``dedicated`` to ``default``. Updating ``InstanceTenancy`` from ``default`` to ``dedicated`` requires replacement.",
"type": "string"
},
```

これを何も考えずにRustコードに変換すると、当然のことながら`string`型になる。

```rust
        .attribute(
            AttributeSchema::new("instance_tenancy", AttributeType::String)
                .with_description("The allowed tenancy of instances launched into the VPC.  + ``default``: An instance launched into the VPC runs on shared hardware by default, unless y..."),
)
```

そこで、コードジェネレーターでdescriptionを解析して取り得る値を抽出し、名前空間付き列挙型に変換するようにしてみた。

こんな感じで、列挙型のバリデーション関数を生成してくれる。

```rust
const VALID_INSTANCE_TENANCY: &[&str] = &["default", "dedicated", "host"];

fn validate_instance_tenancy(value: &Value) -> Result<(), String> {
    validate_namespaced_enum(
        value,
        "InstanceTenancy",
        "awscc.ec2_vpc",
        VALID_INSTANCE_TENANCY,
    )
}
```

そして、スキーマ定義では、カスタム属性型として定義してくれる。

```rust
        .attribute(
            AttributeSchema::new("instance_tenancy", AttributeType::Custom {
                name: "InstanceTenancy".to_string(),
                base: Box::new(AttributeType::String),
                validate: validate_instance_tenancy,
                namespace: Some("awscc.ec2_vpc".to_string()),
            })
                .with_description("The allowed tenancy of instances launched into the VPC.  + ``default``: An instance launched into the VPC runs on shared hardware by default, unless y..."),
        )
```
