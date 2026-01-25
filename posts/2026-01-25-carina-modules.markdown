---
title: Carina Modules
date: 2026-01-25 12:59:31 +0900
---

[昨日の記事](/blog/2026/01/24/1/)で紹介した[Carina](https://github.com/mizzy/carina)に、モジュール機能を実装した。

---

## 背景

[YAPC::Fukuoka 2025での発表](/blog/2025/11/17/1/)で、インフラコードのモジュール化の難しさについて話した。

<script defer class="speakerdeck-embed" data-id="facd81a255fb4c32ab2d80c559f1ede1" data-ratio="1.7777777777777777" src="//speakerdeck.com/assets/embed.js"></script>

ここで挙げた問題のひとつが「内部構造の把握と依存関係の追跡が困難」という点。Terraformのモジュールは中身を見ないと何が作られるかわからないし、依存関係も追いづらい。planを実行してみれば、どんなリソースがつくられるかはわかるが、モジュールを呼び出すためのコードを書かないといけなくて面倒。また、つくられるリソースは表示されるが、リソース間の依存関係はplanの出力からはわからない。

この問題を解決するために、Carinaには`module info`というコマンドを実装した。

---

## module info コマンド

`module info`コマンドを実行すると、モジュールの情報が以下のように表示される。

![module info の出力](/images/2026/01/carina-module-info.png)

3つのセクションに分かれている。

### REQUIRES

モジュールが必要とする入力を表示する。

```
vpc: ref(aws.vpc)  (required)
cidr_blocks: list(cidr)  (required)
enable_https: bool = true
```

型情報と、必須かどうか、デフォルト値があるかどうかがわかる。

### CREATES

モジュールが作成するリソースをツリー構造で表示する。

```
input { vpc: ref(aws.vpc) }
└── web_sg: aws.security_group
    ├── http: aws.security_group.ingress_rule
    └── https: aws.security_group.ingress_rule
```

入力として受け取った`vpc`を起点に、どのリソースがどのリソースに依存しているかが視覚的にわかる。

### EXPOSES

モジュールが外部に公開する出力を表示する。

```
security_group: ref(aws.security_group)
  ← from: web_sg
```

出力の型と、その出力がモジュール内のどのリソースから来ているかがわかる。

---

## 今後

まだ少ないリソース数と単純な依存関係でしか試していないが、モジュールで定義されているリソース構造を把握するのに役立ちそう。リソース数が増えたり、依存関係が複雑になったりするとどうなるか、引き続き試しながら改善していきたい。
