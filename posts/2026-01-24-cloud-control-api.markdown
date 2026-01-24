---
title: AWS Cloud Control APIは遅い？
date: 2026-01-24 18:40:23 +0900
---

[Carina](https://github.com/mizzy/carina)の開発中に、AWS Cloud Control APIが遅いことに気づいた。

---

## Cloud Control APIを試した背景

CarinaでAWSリソースを扱うにあたり、最初はaws-sdk-s3を使ってS3バケットの操作を実装していた。ただ、この方法だとリソースタイプごとに個別実装が必要になる。S3、EC2、IAM...とAWSのサービスは膨大にあるので、全部個別に実装するのは大変だ。生成AIがあるとはいえ、もっと楽にできないかと考えた。

そこで[Cloud Control API](https://docs.aws.amazon.com/cloudcontrolapi/latest/userguide/what-is-cloudcontrolapi.html)を使うことを考えた。Cloud Control APIは、CloudFormationと同じリソースハンドラーを使った統一的なAPIで、様々なAWSリソースをCRUD操作できる。

Cloud Control APIを使う大きなメリットとして、CloudFormationのJSON Schemaが公開されていることがある。例えばS3バケットのスキーマは以下のコマンドで取得できる。

```shell
aws cloudformation describe-type --type RESOURCE \
  --type-name AWS::S3::Bucket --query 'Schema' --output text
```

このスキーマを使えば、[typify](https://github.com/oxidecomputer/typify)でRustの型を自動生成できる。生成した型を使ってバリデーションすれば、リソースごとに手動でバリデーションロジックを書く必要がなくなる。

というわけで、Cloud Control APIを使うよう[実装してみた](https://github.com/mizzy/carina/pull/2)。

---

## 問題発覚

S3バケットの`versioning_configuration`を変更する`apply`を実行したところ、完了までに20秒以上かかった。単純なプロパティ変更なのに遅すぎるのでは、と思って調べてみた。

---

## 計測結果

直接S3 APIを叩いた場合と、Cloud Control APIを叩いた場合で比較してみた。

### 直接S3 API

```shell
$ time aws s3api put-bucket-versioning \
  --bucket carina-test-logs-mizzy-2026 \
  --versioning-configuration Status=Suspended
```

所要時間: **約0.4〜0.5秒**

### Cloud Control API

```shell
RESULT=$(aws cloudcontrol update-resource \
  --type-name "AWS::S3::Bucket" \
  --identifier "carina-test-logs-mizzy-2026" \
  --patch-document '[{"op":"replace","path":"/VersioningConfiguration","value":{"Status":"Enabled"}}]')

REQUEST_TOKEN=$(echo "$RESULT" | jq -r '.ProgressEvent.RequestToken')
START=$(date +%s)

while true; do
  STATUS=$(aws cloudcontrol get-resource-request-status \
    --request-token "$REQUEST_TOKEN" | jq -r '.ProgressEvent.OperationStatus')
  echo "[$(( $(date +%s) - START ))s] $STATUS"
  [ "$STATUS" = "SUCCESS" ] || [ "$STATUS" = "FAILED" ] && break
  sleep 1
done
```

完了までの所要時間: **約23秒**

---

## なぜ遅いのか

Cloud Control APIはCloudFormationと同じリソースハンドラーを使っており、内部的にリソースの整合性チェックや安定化待ち（resource stabilization）を行っているようだ。

AWSの公式ブログ[How we sped up AWS CloudFormation deployments with optimistic stabilization](https://aws.amazon.com/blogs/devops/how-we-sped-up-aws-cloudformation-deployments-with-optimistic-stabilization/)でも、CloudFormationのデプロイが遅い理由として「resource stabilization」に時間がかかることが説明されている。

---

## まとめ

| 方式 | 所要時間 |
|------|----------|
| 直接S3 API | 約0.5秒 |
| Cloud Control API | 約23秒 |

Cloud Control APIは統一的なインターフェースで様々なリソースを扱えるメリットがあるが、個別のAPIを直接叩くより大幅に遅い。速度が重要な場合は、リソースタイプごとに直接SDKを使う方がよさそう。

Carinaでの採用はやめた。
