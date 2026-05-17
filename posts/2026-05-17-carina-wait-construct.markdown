---
title: Carinaのwait構文
date: 2026-05-17 18:08:30 +0900
---

[Carina](https://github.com/carina-rs/carina)に`wait`という構文を入れた。「あるリソースが特定の状態になるまで、それに依存するリソースの作成をブロックする」ための仕組みで、Terraformでいうと`aws_acm_certificate_validation`のような「待ち合わせ専用リソース」に相当する。

現在構築中のCarina provider registry（Carina自身で管理している）でCloudFront + ACMの構成を組んでいて、ACM証明書がISSUEDになるまでCloudFrontの作成を待ちたい、というのが直接のきっかけ。Terraform流の「待ち合わせリソース」をproviderに足すアプローチも検討したけど、結局DSL側に`wait`構文を入れる方向にした。その経緯と設計について書いておく。

---

## ACM + CloudFrontでの待ち合わせ問題

ACMでDNS検証を使ってCloudFront用の証明書を発行する場合、流れはこうなる。

1. ACM証明書をリクエストする
2. 証明書のレスポンスに含まれる検証情報から、検証用のCNAMEレコードの内容を取得する
3. Route 53にその検証用CNAMEレコードを作る
4. ACMがそのレコードを観測して証明書のステータスを`PENDING_VALIDATION` → `ISSUED`に遷移させる
5. CloudFrontなど下流のリソースが`ISSUED`になった証明書のARNを使い始める

問題は4と5の間で、証明書がまだ`ISSUED`になっていないうちにCloudFrontの作成が走ってしまうケース。CloudFrontはviewer certificateに`ISSUED`済みの証明書しか受け付けないので、`PENDING_VALIDATION`のままの証明書を渡すと作成自体が失敗する。なので「証明書が`ISSUED`になるまでCloudFrontの作成を止める」という同期点が必要になる。

Terraformでは`aws_acm_certificate_validation`という専用のリソースを介してこれを実現している。

```hcl
resource "aws_acm_certificate" "cert" {
  domain_name       = "registry.carina-rs.dev"
  validation_method = "DNS"
}

resource "aws_route53_record" "validation" {
  zone_id = aws_route53_zone.zone.id
  name    = tolist(aws_acm_certificate.cert.domain_validation_options)[0].resource_record_name
  type    = tolist(aws_acm_certificate.cert.domain_validation_options)[0].resource_record_type
  records = [tolist(aws_acm_certificate.cert.domain_validation_options)[0].resource_record_value]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "cert" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [aws_route53_record.validation.fqdn]
}

resource "aws_cloudfront_distribution" "dist" {
  # ...
  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate_validation.cert.certificate_arn
    # 直接 aws_acm_certificate.cert.arn を参照すると ISSUED 前に作成が走る
  }
}
```

下流のCloudFrontは`aws_acm_certificate.cert.arn`ではなく、`aws_acm_certificate_validation.cert.certificate_arn`を参照する。`aws_acm_certificate_validation`が「ISSUEDになるまでブロックする」という役割を担っている。

ただ、この`aws_acm_certificate_validation`という設計には少し違和感がある。

- AWS側に対応する実リソースが無い。`aws_acm_certificate_validation`という名前のものがAWSに作られるわけではなく、実体は「証明書がISSUEDになるまで待つ」という処理でしかない。「リソース」と書いてあるのに実体がないので、初見だと何をしているリソースなのか分かりにくい
- 同じパターンが他のサービスでも繰り返される。RDSの`available`待ち、EC2の`running`待ち、Lambdaの`Active`待ち、など、各providerに似たような「待ち合わせ専用リソース」が並ぶことになる

これは`aws_acm_certificate_validation`に限った話ではなく、Terraformでは「実体のない繋ぎ・待ち合わせ用のリソース」というパターンが広く使われている。`time_sleep`（指定秒数待つ）、`null_resource`、`terraform_data`あたりが代表例で、いずれもAWS等に対応する実体は無く、依存関係のエッジを足したり処理をフックしたりするためだけに「リソース」の形を借りている。

これ自体は実害があるというより、使う側として「これは何のリソースなんだろう」と一瞬考えさせられる、という類の引っかかり。Terraformのリソースモデルが表現力豊かなので、本来リソースでないものまでリソースの形で書けてしまう、とも言える。

Carinaでも同じく専用リソース (`aws.acm.CertificateValidation`) を足す手もあったけど、実体のない待ち合わせ処理をリソースとして表現するのは、やはり違和感がある。Terraformと同じ妥協を後発のCarinaでわざわざ持ち込まなくてもいいのでは、と考えて、「待ち合わせ」をDSLの第一級構文として導入することにした。設計の詳細は[notes/specs/2026-05-09-wait-construct-design.md](https://github.com/carina-rs/carina/blob/main/notes/specs/2026-05-09-wait-construct-design.md)に書いてある。

---

## Carinaの`wait`構文

実際の構文はこんな感じ。実際に運用しているACM証明書の定義から抜粋。

```crn
let cert = aws.acm.Certificate {
  domain_name       = domain_name
  validation_method = dns
}

let validation_record = aws.route53.RecordSet {
  hosted_zone_id   = zone.id
  name             = cert.domain_validation_options[0].resource_record.name
  type             = cname
  ttl              = 300
  resource_records = [cert.domain_validation_options[0].resource_record.value]
}

let cert_issued = wait cert {
  until      = cert.status == aws.acm.Certificate.Status.issued
  depends_on = [validation_record]
  timeout    = 75min
}
```

`let <name> = wait <target> { ... }`という形で、ターゲットとなるリソースのbinding名 (`cert`) を`wait`の直後に置く。ブロックの中には待ち合わせの挙動を指定する属性を書く。

- `until` — `read()`の結果がこの述語を満たすまで待つ
- `depends_on` — pollを始める前に完了させておきたいbinding。任意。指定しなくても`until`が満たされるまで待つので最終結果は変わらないが、ACMの場合、検証レコードが作られるまではどう頑張ってもISSUEDにならない。その間pollし続けても無駄だし、`timeout`のカウントも進んでしまう。検証レコードの作成完了を待ってからpollを始めるよう、`depends_on = [validation_record]`を指定しておく
- `timeout` — 待ち時間の上限。省略するとリソースのスキーマで宣言されたデフォルトが使われる。ACM Certificateは75分にしてある。これは特に深い根拠があるわけではなく、Terraformの`aws_acm_certificate_validation`のデフォルトタイムアウトが75分なので、それに揃えただけ

下流のCloudFrontは`cert.certificate_arn`ではなく`cert_issued.certificate_arn`を参照する。

```crn
let distribution = awscc.cloudfront.Distribution {
  distribution_config = {
    # ...
    viewer_certificate = {
      acm_certificate_arn      = cert_issued.certificate_arn
      ssl_support_method       = sni_only
      minimum_protocol_version = tlsv1_2_2021
    }
  }
}
```

`cert_issued`は値としては`cert`のpassthroughで、`cert_issued.certificate_arn`は`cert.certificate_arn`と同じ文字列を返す。違うのは依存関係のエッジだけで、`cert_issued`を参照したリソースは「`cert`が`ISSUED`になるまでブロックされる」という同期契約を継承する。

---

## providerには手を入れない

`wait`の実装はすべてcarina-core側（executor）で行う。providerには手を入れていない。

executorは`wait`エフェクトを以下の手順で処理する。

1. `depends_on`に指定されたエフェクト（ターゲットの`Create`/`Update`、ユーザー指定の追加依存）が完了するまで待つ
2. `provider.read(target_id, ...)`を呼ぶ
3. 返ってきた`State.attributes`に対して`until`述語を評価する
4. 真なら成功。そのときの`State`スナップショットを下流参照用にキャプチャする
5. 偽なら、経過時間が`timeout`を超えていればタイムアウトエラー、超えていなければ`interval`だけ`sleep`して2に戻る

providerに対して呼ばれるのは普通の`read()`だけで、provider側は`wait`の存在を知らない。Terraformのように待ち合わせ専用のリソースをproviderごとに実装しなくても、どのproviderのどのリソースでも`wait`できる、というのが地味にうれしいポイント。

---

## まとめ

Terraformの`aws_acm_certificate_validation`とCarinaの`wait`の違いを並べるとこんな感じ。

| Terraformの`aws_acm_certificate_validation` | Carinaの`wait` |
|---|---|
| providerごとに専用リソース (`~Validation`、`~Ready` など) を足す必要がある | DSLの第一級構文。どのproviderのどのリソースにも使える |
| `aws_acm_certificate_validation`の作成処理が同期するため、providerの責務にwait処理が入る | wait処理はexecutor側にあり、providerはwaitしない |

ACM以外にも、EC2の`running`待ち、RDSの`available`待ち、Lambdaの`Active`待ち、など同じパターンが他のサービスでも使える。
