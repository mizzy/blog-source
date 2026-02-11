---
title: Carina awscc providerのacceptance test
date: 2026-02-11 21:27:18 +0900
---

[Carina](https://github.com/carina-rs/carina)のawscc providerにacceptance testを追加した。実際にAWSリソースを作成・削除して、プロバイダーが正しく動作するかを確認するテスト。

Carinaのプロバイダーのコードは全部生成AIにつくってもらっているので、網羅的なテストが重要になる。自分の手でコードを書いていれば、ある程度の品質は把握できるが、生成AIが書いたコードはちゃんとテストしないと信用できない。一方で、こういった地道なacceptance testを手作業で整備・実行・確認するのは正直面倒なので、テストの生成、実行、確認も全部生成AIにやってもらっている。コードを書くのも生成AI、テストを書くのも生成AI、テストを実行して確認して見つかったバグを直すのも生成AI、という構図。

---

## テストの構成

テストは`.crn`ファイルとして記述する。リソースタイプごとにディレクトリを分け、パターンごとにファイルを用意している。現時点で20のEC2リソースタイプに対して39のテストファイルがある。

```
acceptance-tests/
├── ec2_vpc/
│   ├── basic.crn
│   └── with_ipam.crn
├── ec2_subnet/
│   ├── basic.crn
│   ├── ipv6.crn
│   ├── with_dns_options.crn
│   └── with_ipam.crn
├── ec2_route/
│   ├── igw.crn
│   ├── ipv6.crn
│   ├── nat_gateway.crn
│   ├── transit_gateway.crn
│   └── vpc_peering.crn
├── ...
└── run-tests.sh
```

テストファイルの中身は通常の`.crn`ファイルと同じ。たとえば`ec2_vpc/basic.crn`はこんな感じ。

```crn
# EC2 VPC - Basic test
#
# Tests: cidr_block, enable_dns_support, enable_dns_hostnames, instance_tenancy, tags

provider awscc {
  region = aws.Region.ap_northeast_1
}

awscc.ec2_vpc {
  name                 = "acceptance-test-vpc-basic"
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  instance_tenancy     = default

  tags = {
    Environment = "acceptance-test"
  }
}
```

テストといっても特別なフレームワークは使っておらず、Carinaの`.crn`ファイルをそのまま使っている。このファイルに対して`validate`→`apply`→`destroy`を実行して、すべて成功すればOK。

---

## テストランナー

`run-tests.sh`というシェルスクリプトでテストを実行する。以下のコマンドをサポートしている。

```bash
# AWSクレデンシャル不要
./run-tests.sh validate

# aws-vaultで単一アカウント実行
aws-vault exec myprofile -- ./run-tests.sh plan
aws-vault exec myprofile -- ./run-tests.sh apply
aws-vault exec myprofile -- ./run-tests.sh destroy

# 10アカウント並列でapply+destroyサイクル
./run-tests.sh full
```

`validate`はAWSクレデンシャル不要で、DSLの構文チェックのみ。`plan`/`apply`/`destroy`は単一アカウントに対して実行する。

---

## 10アカウント並列実行

`full`コマンドは、10個のAWSアカウント（`carina-test-000`〜`carina-test-009`）を用意して、テストをラウンドロビンで各アカウントに分配し、並列でapply+destroyサイクルを実行する。

```bash
./run-tests.sh full
```

を実行すると、39個のテストが10アカウントに分配され、各アカウントで4個前後のテストが並列に走る。テストごとにリソースを作成して、正常に作成できたことを確認したらすぐに削除する。

10アカウントを用意している理由は、テスト時間の短縮と、テスト間の独立性の確保。同じアカウントで多くのリソースを同時に作成・削除すると、AWSのAPI rate limitやリソースクォータに引っかかったり、リソース間の依存関係で問題が起きたりする。アカウントを分けることでこれらを回避している。

---

## テストで見つかったバグ

このacceptance testを実行してみて、いくつかバグが見つかった。

- [#75](https://github.com/carina-rs/carina/issues/75) - `apply`で一部リソースの作成に失敗しても終了コード0を返していた
- [#76](https://github.com/carina-rs/carina/issues/76) - IPAM Poolの削除がCloud Control API経由だとタイムアウトする
- [#77](https://github.com/carina-rs/carina/issues/77) - IPAM PoolのCIDR伝播遅延でサブネット割り当てが失敗する
- [#80](https://github.com/carina-rs/carina/issues/80) - 削除に時間がかかるリソースがあると、依存リソースの削除順序が崩れる
- [#81](https://github.com/carina-rs/carina/issues/81) - `ipv4_ipam_pool_id`が`AwsResourceId`型になっていた（正しくは`IpamPoolId`型）

単体テストだけでは見つけられなかったバグが結構出てきたので、やはり実際のAWSに対するテストは重要。
