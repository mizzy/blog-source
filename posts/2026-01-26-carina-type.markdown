---
title: Carinaの型システムについて
date: 2026-01-27 09:54:37 +0900
---
Terraformライクなツール[Carina](https://github.com/mizzy/carina)をつくりはじめた動機のひとつに、「型システムの強化」がある。

Terraformではプリミティブな型（string, number, bool）に加えて、listやmap、objectなどの複合型がサポートされている。しかし、型システム自体はあまり強力ではない。

例えば、regionやcidr_blockのような特定のドメインに特化した型は存在しない。そのため、これらの値を扱う際には単なるstring型として扱われ、誤った値が設定される可能性がある。誤った設定をした場合、validateやplanで気づけるのであればまだ良いが、planが通ってしまい、apply時にエラーになってはじめて間違いが発覚することもある。

これを防ぐために、Carinaではドメイン固有型の導入を試している。現在実装しているものとして、region型やcidr型がある。これらの型は、それぞれのドメインに特化したバリデーションロジックを持っており、不正な値が設定された場合には即座にエラーを返す。

例えば、誤ってリージョンを指定すべきところにアベイラビリティゾーンを指定したとする。

```hcl
provider aws {
  region = "ap-northeast-1a"
}
```

Terraformではさすがに`plan`は通らないが、`validate`は通る。

```shell
$ terraform validate
Success! The configuration is valid.
```

Carinaもproviderの定義はTerraformと同じように記述するが、`validate`で以下のようなエラーが返る。

```shell
Error: provider aws: Validation failed: Invalid region 'ap-northeast-1a', expected one of: ap-northeast-1, ap-northeast-2, ap-northeast-3, ap-southeast-1, ap-southeast-2, ap-south-1, us-east-1, us-east-2, us-west-1, us-west-2, eu-west-1, eu-west-2, eu-west-3, eu-central-1, eu-north-1, ca-central-1, sa-east-1 or DSL format like aws.Region.ap_northeast_1
```

このようにCarinaでは、型システムを強化することで、より早い段階で設定ミスを検出できるようにしたい、と考えている。

また、CarinaではAWSリージョンはEnum型として定義されているため、以下のようにリージョンを指定することもできる。

```crn
provider aws {
  region = aws.Region.ap_northeast_1
}
```

更にLSPもサポートしているため、IDE上で補完候補としてリージョンの一覧が表示される。

![リージョンの補完候補](/images/2026/01/carina-region-completion.png)

誤った文字列を指定している場合も、IDE上でエラーが表示される。

![IDEでのエラー表示](/images/2026/01/carina-region-error.png)

もう一つの例として、cidr型を紹介する。cidr型はCIDR表記のIPアドレスブロックを表す型で、不正なCIDR表記が指定された場合にはエラーを返す。

例えば、以下のように不正なプレフィックス長を指定したとする。

```crn
let main_vpc = aws.vpc {
    name       = "main-vpc"
    cidr_block = "10.0.0.0/33"
}
```

`carina validate`を実行すると、以下のようなエラーが返る。

```
Error: vpc.main-vpc: Validation failed: Invalid prefix length '33': must be 0-32
```

Terraformでもvalidateで同様のエラーが返るが、IntelliJ IDEAのTerraformプラグインではCIDR表記のフォーマットエラーが検出されない。CarinaのLSPでは、IDE上で以下のようにエラーが表示される。

![cidr_blockのエラー表示](/images/2026/01/carina-cidr-error.png)

まったくCIDR表記として成り立っていない場合も、当然ながらIDE上でエラーが表示される。

![cidr_blockのフォーマットエラー表示](/images/2026/01/carina-cidr-format-error.png)

cidr型でもregion型と同様に、補完候補として一般的なCIDR表記が表示される。

![cidr_blockの補完候補](/images/2026/01/carina-cidr-completion.png)

さらに別の例として、S3バケットの`versioning_configuration`属性を紹介する。`versioning_configuration`はS3バケットのバージョニング設定を表すオブジェクト型で、`status`フィールドには`Enabled`または`Suspended`のいずれかを文字列で指定する必要がある（`Disabled`もあるがこれはちょっと特殊な値）。誤った文字列を指定すると、validateやplanでは不正な値が検出されるが、IDE上ではエラーや補完候補が表示されないため、あれ、Enableだっけ？Enabledだっけ？と迷ったら公式ドキュメントを確認する必要がある。

```hcl
resource "aws_s3_bucket_versioning" "versioning_example" {
  bucket = aws_s3_bucket.example.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

Carinaでは、このstatusフィールドをEnum型として定義しているため、以下のように補完候補として"Enabled"と"Suspended"が表示される。

![S3バケットのversioning補完](/images/2026/01/carina-s3-versioning-completion.png)

エディタ上でのエラー表示や自動補完については、今は生成AIが基本的にコードを書いてくれるのであまり気にする必要はないかもしれない。しかし、型システムが強力であればあるほど、生成AIがより正確なコードを生成できるようになり、早い段階で間違いに気づけるようになるはず。

型システムを強化するためには、各リソースの属性やその型情報を正確に把握する必要がある。AWSの場合、CloudFormationのリソースタイプスキーマを利用することで、各リソースの属性情報を取得できる。例えば、以下のコマンドを実行すると、S3バケットリソースのスキーマ情報が取得できる。

```shell
aws cloudformation describe-type \
      --type RESOURCE \
      --type-name AWS::S3::Bucket \
      --query "Schema" \
      --output text
```

これを元にCarinaで型定義を自動生成するなり、AIに生成させるなりすれば、型システムの強化がより容易になると考えている。
