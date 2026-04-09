---
title: Carina aws providerとawscc providerのベンチマーク
date: 2026-04-09 14:51:03 +0900
---

[Carina](https://github.com/carina-rs/carina)のaws provider（SDK直接呼び出し）とawscc provider（Cloud Control API）のパフォーマンスを比較した。

以前、[AWS Cloud Control APIは遅い？](https://mizzy.org/blog/2026/01/24/2/)という記事で、Cloud Control APIが直接SDKを叩くより大幅に遅いことを書いた。あの時点では「Carinaでの採用はやめた」と書いたが、その後、統一的なインターフェースのメリットからawscc providerとして復活させた。今回は、両方のproviderが揃ったので、実際のCarina上でのベンチマークを取った。

---

## 計測環境

- リージョン: ap-northeast-1
- 各操作3回実行の平均値
- Carinaのオーバーヘッド（パース、diff、state I/O）を含む

---

## 単一リソースの比較

| リソース | 操作 | aws | awscc | 倍率 |
|----------|------|-----------|------------|------|
| S3 Bucket | Create | 2.0s | 16.0s | 7.7x |
| S3 Bucket | Read | 376ms | 921ms | 2.4x |
| S3 Bucket | Destroy | 900ms | 6.1s | 6.8x |
| EC2 VPC | Create | 1.1s | 16.1s | 13.8x |
| EC2 VPC | Read | 382ms | 578ms | 1.5x |
| EC2 VPC | Destroy | 973ms | 5.7s | 5.9x |
| EC2 Security Group | Create | 1.9s | 26.5s | 13.4x |
| EC2 Security Group | Read | 434ms | 712ms | 1.6x |
| EC2 Security Group | Destroy | 1.1s | 10.8s | 9.5x |

単一リソースだと、以前の記事と同じ傾向。Createで8-14倍、Destroyで6-10倍、Readで1.5-2.4倍の差がある。Cloud Control APIのresource stabilization（リソースの安定化待ち）が主な原因。

---

## 複数リソースの比較

単一リソースの比較だけだと実運用のパフォーマンスはわからない。実際のインフラは複数のリソースが依存関係を持って構成される。そこで、VPC + サブネット + ルートテーブル + NAT Gateway + セキュリティグループなど、16リソースで構成されるVPCまわり一式でもベンチマークを取った。

| 操作 | aws | awscc | 倍率 |
|------|-----------|------------|------|
| Apply（16リソース） | 130.2s | 152.5s | 1.2x |
| Plan（全リソースread） | 0.7s | 1.4s | 2.0x |
| Destroy（16リソース） | 57.9s | 69.3s | 1.2x |

単一リソースでは8-14倍の差があったのが、16リソースになると1.2倍まで縮まった。

---

## なぜ差が縮まるのか

Carinaは依存関係のないリソースを並列に実行する。16リソースのうち、互いに依存しないものは同時にAPIを叩く。すると、ボトルネックは個々のAPI呼び出しの速度ではなく、依存チェーンの中で最も遅いリソースになる。この構成ではNAT Gatewayの作成（約90秒）がボトルネックで、これはawsでもawsccでも同じ。

つまり、並列実行が個々のAPIの遅さを吸収してしまう。

---

## まとめ

- 単一リソースではaws providerが圧倒的に速い（8-14倍）
- 複数リソースの実運用シナリオでは差が1.2倍まで縮まる
- `carina plan`はReadが主なので、aws providerが1.5-2倍速い

単一リソースの数字だけ見るとawscc providerは使い物にならないように見えるが、実運用では並列実行のおかげで差はかなり小さい。awscc providerはCloudFormationスキーマからの自動生成でカバレッジが広いというメリットがあるので、パフォーマンスだけで選ぶ必要はない。
