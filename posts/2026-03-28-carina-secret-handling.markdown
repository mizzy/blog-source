---
title: Carinaのシークレット管理
date: 2026-03-28 14:52:19 +0900
---

[Carina](https://github.com/carina-rs/carina)にシークレット管理の仕組みを設計・実装しているので、Terraformとの設計の違いについて書いておく。

---

## Terraformのシークレット問題

Terraformにおけるシークレット管理は長年の課題だった。[GitHub issue #516](https://github.com/hashicorp/terraform/issues/516)は2014年10月に作成され、250以上のコメントがついている。

問題の根本は、Terraformのstateにシークレットが平文で保存されることにある。`sensitive`フラグをつけても、plan出力がマスクされるだけで、stateには平文のまま残る。`aws_kms_secrets`データソースを使っても、復号結果はstateに平文で保存される。

この問題に対してTerraformは段階的に対処してきた。

- **sensitive**（Terraform 0.14、2020年）: plan出力のマスク。stateは平文のまま
- **ephemeral**（Terraform 1.10、2024年11月）: stateに一切保存しない値。`ephemeral`リソースやephemeral変数として使う
- **write-only**（Terraform 1.11、2025年3月）: リソース属性の値をstateに保存しない仕組み

問題提起から約10年かけて、ようやく「stateに保存しない」という本質的な解決策にたどり着いた形になる。

---

## Terraformのwrite-only

write-onlyはTerraform 1.11で導入された仕組みで、リソースの属性値をstateに一切保存しないようにする。例えばSSM Parameterにシークレットを保存する場合、従来は値がstateに平文で残っていたが、write-onlyを使うとこうなる。

```hcl
data "aws_kms_secrets" "example" {
  secret {
    name    = "db_password"
    payload = "AQICAHh..."
  }
}

resource "aws_ssm_parameter" "db_password" {
  name             = "/myapp/db_password"
  type             = "SecureString"
  value_wo         = data.aws_kms_secrets.example.plaintext["db_password"]
  value_wo_version = 1
}
```

stateに値を保存しないので安全だが、代わりに変更検知ができなくなるという問題がある。そこで`value_wo_version`という数値を手動で管理する。値を変更したらこれを1から2にインクリメントする。Terraformはこのバージョン番号の変更を検知して、プロバイダーに新しい値を送る。

実際にversionを変更してplanすると、こうなる。

```hcl
  # aws_ssm_parameter.db_password will be updated in-place
  ~ resource "aws_ssm_parameter" "db_password" {
      ~ has_value_wo     = true -> (known after apply)
        id               = "/tf-write-only-test/db_password"
        name             = "/tf-write-only-test/db_password"
      ~ value_wo_version = 1 -> 2
        # (12 unchanged attributes hidden)
    }
```

`value_wo_version`の変更は表示されるが、`value_wo`自体の値や変更内容は一切表示されない。

この設計にはいくつかの問題がある。

- バージョンのインクリメントを忘れると、実際の値が変わっていてもTerraformは変更を検知しない
- リソースタイプごとに個別にwrite-only対応が必要で、対応はまだ一部のリソースに限られる
- `_wo`サフィックスの命名規則により、既存の属性名とは別の属性が必要になる

なお、[プラグインフレームワークのドキュメント](https://developer.hashicorp.com/terraform/plugin/framework/resources/write-only-arguments)では、private stateにハッシュを保存して変更を検知するアプローチもベストプラクティスとして紹介されている。ただし、少なくともAWS providerでは現時点ではこの方式は採用されていないようだ。

---

## Carinaのsecret() + decrypt()

Carinaでは同じことを`secret()`（[#1238](https://github.com/carina-rs/carina/issues/1238)）と`decrypt()`（[#1240](https://github.com/carina-rs/carina/issues/1240)）の組み合わせで実現する。

```crn
awscc.ssm.parameter {
  name  = "/myapp/db_password"
  type  = "SecureString"
  value = secret(decrypt("AQICAHh..."))
}
```

`decrypt()`はapply時にプロバイダーの暗号化サービス（AWS KMS等）を使って復号する。`secret()`で包むことで、復号後の値はstateにハッシュとしてのみ保存される。

`secret()`の動作は以下の通り。

- **apply時**: `secret()`をアンラップして実際の値をプロバイダーに送る
- **state**: 値そのものではなくArgon2idハッシュのみを保存する。ソルトはリソースタイプ・リソース名・属性キーから生成するので、同じ値でもリソースごとに異なるハッシュになる
- **plan出力**: 値の代わりに`(secret)`と表示する。変更時は`(secret) → (secret)`と表示され、値は見えないが変更があったことはわかる
- **diff**: ハッシュの比較で変更を自動検知する

Terraformのwrite-onlyとの違いをまとめると以下の通り。

- ハッシュをstateに保存することで変更検知を自動化。バージョン番号を手動で管理する必要がない
- ハッシュから元の値を復元することはできないので、stateからシークレットが漏洩するリスクもない
- `secret()`は組み込み関数なので、プロバイダー側の個別対応は不要。どのリソースのどの属性にも使える
- Terraformでは復号用のデータソースを別途定義する必要があるが、Carinaでは属性値に直接書ける
- `decrypt()`はプロバイダー非依存の設計で、AWS KMS、GCP Cloud KMS、Azure Key Vaultなどプロバイダーの暗号化サービスに自動的に委譲する予定

---

## env()との組み合わせ

暗号文を`.crn`ファイルに書きたくない場合は、`env()`関数（[#1239](https://github.com/carina-rs/carina/issues/1239)、実装済み）で環境変数から渡すこともできる。

```crn
awscc.rds.db_instance {
  master_password = secret(env("DB_PASSWORD"))
}
```

---

## 設計の違いまとめ

Carinaのシークレット管理は、`secret()`と`decrypt()`の2つの関数で実現する。

- `secret(value)` — 値をシークレットとしてマーク。stateにはハッシュのみ保存
- `decrypt(ciphertext)` — プロバイダーの暗号化サービスで復号

これらに`env()`のような汎用関数を組み合わせることもできる。それぞれが単一の責務を持ち、自由に組み合わせられる。Terraformでは`sensitive`、`ephemeral`、write-only、`aws_kms_secrets`データソースと、歴史的経緯から複数の仕組みが並立している。Carinaは後発の利点を活かして、最初からシンプルな設計にできた、と思う。

ただし、これはあくまで現段階の設計であり、より安全で良い方法が見つかれば変更する可能性はある。
