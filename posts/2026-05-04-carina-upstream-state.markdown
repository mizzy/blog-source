---
title: Carinaのupstream state機能
date: 2026-05-04 08:47:03 +0900
---

[Carina](https://github.com/carina-rs/carina)に`upstream_state`という機能を入れた。別のCarinaコンポーネントのstateを参照するための仕組みで、Terraformでいう`terraform_remote_state`に相当する。

ただ、Terraformのremote stateを使っていて気になる点があったので、Carinaでは少し違う設計にしてみた。その違いについて書いておく。

---

## Terraformのremote stateのおさらい

Terraformで別コンポーネントのstateを参照するときは、`terraform_remote_state` data sourceを使う。

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tfstate"
    key    = "network/terraform.tfstate"
    region = "ap-northeast-1"
  }
}

resource "aws_security_group" "web" {
  vpc_id = data.terraform_remote_state.network.outputs.vpc_id
}
```

参照元のoutputsを`data.terraform_remote_state.network.outputs.vpc_id`のような形で読む。参照される側は`output`ブロックで値を公開する。

使っていて気になる点がいくつかある。

### 参照する側がbackendの設定を知っている必要がある

networkコンポーネントのbackendがS3なのか、バケット名は何か、キーは何か、といった情報を、それを使うweb側のコードにも書く必要がある。参照元のnetwork側ではこう書いて、

```hcl
# network/main.tf
terraform {
  backend "s3" {
    bucket = "my-tfstate"
    key    = "network/terraform.tfstate"
  }
}
```

参照する側のweb側ではこう書く。

```hcl
# web/main.tf
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tfstate"
    key    = "network/terraform.tfstate"
  }
}
```

bucketとkeyが両方に登場しているのが分かる。実質的に同じ情報を二重管理している状態になる。

### 呼び出すのに長い文字列を書く必要がある

参照するときの記述が`data.terraform_remote_state.network.outputs.vpc_id`となって、なかなか長い。`data.terraform_remote_state.`と`.outputs.`が毎回ついてくる。

複数の値を参照していると、この長い名前があちこちに出てきて冗長になる。localsで別名をつけて短くする、みたいな回避策を取ることになる。

### 補完が効かない

`outputs`の中身は実際のstateを読まないと分からないので、エディタで補完が効かない。

補完が効かないので、typoしても実行するまで気付けない。`terraform validate`は実際のremote stateを読まずに構文チェックだけするので、ここでは検出されない。別コンポーネントで`output`の名前を変えたときに、参照側が壊れていることを`plan`や`apply`のタイミングで初めて知ることになる。

### 参照元が未applyだとplanできない

共通の通知基盤となるSNS topicをmonitoringコンポーネントで管理し、各サービス側のCloudWatchアラームからそのtopicを参照する、という構成を考える。

monitoring側に新しい通知先（たとえば`critical_alerts_topic_arn`）を追加し、同じPRでweb側の新しいアラームからそれを参照する、というPRを出したとする。

```hcl
# monitoring/main.tf
resource "aws_sns_topic" "critical_alerts" {
  name = "critical-alerts"
}

output "critical_alerts_topic_arn" {
  value = aws_sns_topic.critical_alerts.arn
}
```

```hcl
# web/main.tf
resource "aws_cloudwatch_metric_alarm" "web_5xx" {
  alarm_name  = "web-5xx"
  metric_name = "5XXError"
  # ...

  alarm_actions = [
    data.terraform_remote_state.monitoring.outputs.critical_alerts_topic_arn,
  ]
}
```

このPRをCIで`plan`にかけると、monitoring側がまだ`apply`されていないので新しいoutputがstateに存在せず、web側の`plan`はエラーになる。

planエラーを回避したい場合、以下のように`try()`でフォールバックする、という手もある。

```hcl
alarm_actions = try(
  [data.terraform_remote_state.monitoring.outputs.critical_alerts_topic_arn],
  [],
)
```

ただ、これだとPRレビュー時に見えるplanは`alarm_actions = []`になり、monitoringを`apply`した後の本来のplanとは異なる。

結局、monitoring側のPRとweb側のPRを2回に分けてマージする、という対応になりがち。

そもそもremote stateで参照するほど密に依存しているなら、コンポーネントを分割している設計自体を見直したほうがいいケースもある。ただ、stateを分けざるを得ない事情（チーム境界、applyの責任範囲、リソースへのアクセス権限を分けたい都合など）もあり、そういう場合はこの面倒さと付き合うしかない。

remote stateの代わりに、data sourceで名前やタグから引く、という方法もある。これだとremote stateのように「outputが追加されたか」は気にしなくていい。ただし結局、参照先のリソースが実在していないとdata sourceの読み込みに失敗するので、upstreamを先に`apply`する必要があるのは変わらない。

さらにdata sourceはstateではなく実際のAWS APIを叩いてリソース情報を取得するので、参照が増えるとstate refreshが遅くなる。どちらを取るかは悩ましい。

---

## Carinaのupstream_state

Carinaでは、参照する側はupstreamコンポーネントの**ディレクトリを指すだけ**にした。

```crn
let network = upstream_state {
  source = '../network'
}

awscc.ec2.SecurityGroup {
  group_description = 'Web security group'
  vpc_id            = network.vpc_id

  tags = {
    Name = 'web-sg'
  }
}
```

`source`はupstreamコンポーネントのディレクトリで、`.crn`ファイルからの相対パスで解決される。`let`で束縛した名前（この例では`network`）を通じて、upstream側が公開している値にアクセスできる。

ディレクトリ参照なので、別リポジトリにあるコンポーネントを参照するにはsubtreeで取り込むなどの工夫がいる。自分にはリポジトリをまたいでstateを参照したいニーズがないので、いまはこれで割り切っている。必要になったらそのとき考える。

公開する側はこんな感じ。

```crn
backend s3 {
  bucket = 'my-carina-state'
  key    = 'network/carina.state.json'
}

let main = awscc.ec2.Vpc {
  cidr_block = '10.0.0.0/16'
}

exports {
  vpc_id = main.vpc_id
}
```

`exports`ブロックで公開したい値を宣言する。Terraformの`output`ブロックに相当する。型は自動的に推論されるので、この例では`vpc_id`は`VpcId`型として公開される。

---

## Terraformと違うところ

先に挙げた4つの点が、それぞれ`upstream_state`ではどうなっているかを書いていく。

### backend設定を書かなくていい

参照する側に書くのは`source`（ディレクトリ）だけ。backend設定は書かない。

```crn
# web/main.crn
let network = upstream_state {
  source = '../network'
}
```

Carinaは`source`で指定されたディレクトリの`.crn`ファイルを読み込み、そこに書かれている`backend`ブロックを解決し、stateを読み出す。

```crn
# network/main.crn
backend s3 {
  bucket = 'my-carina-state'
  key    = 'network/carina.state.json'
}
```

つまり、backendの場所を知っているのはupstream側だけで、参照する側はそれを意識しなくていい。

ディレクトリ参照なので、LSPでパス補完が効く。さらに、存在しないパスを指定した場合は`carina validate`の時点でエラーになる。エディタ上でも赤線で知らせてくれる。

いっぽうTerraformの`terraform_remote_state`の`config`は、bucket名やkeyをtypoしていても構文的には通るので、`plan`を実行してbackendを叩きに行くまで間違いに気付けない。

### 呼び出しが短い

`let network = upstream_state { source = '../network' }`と束縛しておけば、参照は`network.vpc_id`で済む。`data.terraform_remote_state.`や`.outputs.`のような定型的な部分がない。

`let`で束縛する名前は自由につけられるので、短くしたければ`net`でも何でもいい。

### 補完が効く

`upstream_state`は`source`でupstreamのディレクトリを指しているので、Carinaはupstreamの`.crn`ファイルを直接読める。stateを読まずとも、`exports`ブロックの宣言だけで、何が公開されているかが分かる。

さらにCarinaのDSLは静的な型を持っているので、`exports`で公開される値にも型がつく。型がついていれば、参照側で必要としている型と照合できる。

たとえば参照側で`awscc.ec2.SecurityGroup`の`vpc_id`（型は`VpcId`）を埋めようとすると、同じ`VpcId`型を持つupstreamのexportsがLSPの補完候補に上がってくる。

![upstream_stateのexportsがLSPの補完候補に出る様子](/images/2026/05/upstream-state-completion.png)

存在しないexportを参照すればエラーになる。typoしている場合は近い名前の候補も提示される。

![存在しないexportを参照したときのエラー](/images/2026/05/upstream-state-undefined-export.png)

型が合わない場合もエラーになる（`vpc_id: String = main.vpc_id`のように型アノテーションをつけた場合の例）。

![型が合わないときのエラー](/images/2026/05/upstream-state-type-mismatch.png)

未定義のexport参照や型不一致は`carina validate`でもエラーになるので、`plan`や`apply`を待たずに検出できる。`source`パスが存在しない場合も同様に`carina validate`で検出される。

### 参照元が未applyでもplanできる

`carina validate`はstateの内容は見ず、正しい型でexportされているかどうかだけをチェックするので、参照元が未applyでも当然動く。ではplanの場合はどうか。

upstream側のstate自体がまだ存在しない状態でplanを実行すると、こんな出力になる。

<pre><code><span style="color: #d4a017;">Warning: upstream_state 'network': no state found at /path/to/network;
dependent values will display as `(known after upstream apply: ...)`</span>

Refreshing state...
  ✓ awscc.ec2.SecurityGroup.ec2_security_group_39f08cea [0.0s]

Execution Plan:

  + awscc.ec2.SecurityGroup ec2_security_group_39f08cea
      group_description: "Web security group"
      tags:
        Name: "web-sg"
      <span style="color: #00aaaa;">vpc_id: (known after upstream apply: network.vpc_id)</span>
      group_id: (known after apply)
      id: (known after apply)

Plan: 1 to add, 0 to change, 0 to destroy.
</code></pre>

upstream側のstateが無くてもplanは止まらず、参照値は`(known after upstream apply: network.vpc_id)`というプレースホルダで表示される。「この値はupstreamを`apply`すれば確定する」というのが`plan`時点で見える。

upstream側のstateは存在するが、新しいexport値がまだ入っていない（exportを追加してまだ再applyしていない）場合も、Warningが出ないだけで参照値は同じプレースホルダで表示される。

<pre><code>Refreshing state...
  ✓ awscc.ec2.SecurityGroup.ec2_security_group_39f08cea [0.0s]

Execution Plan:

  + awscc.ec2.SecurityGroup ec2_security_group_39f08cea
      group_description: "Web security group"
      tags:
        Name: "web-sg"
      <span style="color: #00aaaa;">vpc_id: (known after upstream apply: network.vpc_id)</span>
      group_id: (known after apply)
      id: (known after apply)

Plan: 1 to add, 0 to change, 0 to destroy.
</code></pre>

Terraformでは参照先のstateが無いとremote stateの読み込みで失敗してplan自体が出せないが、Carinaはプレースホルダで表示するので、planは出せる。前述の`try()`での回避策のように「実際とは違うplan」が表示されるわけでもない。upstreamを`apply`した後にもう一度planを取れば、プレースホルダが実際の値に置き換わるだけ。

---

## まとめ

Terraformのremote stateで気になる点と、Carinaの`upstream_state`での対応を並べるとこんな感じ。

| Terraformのremote state | Carinaの`upstream_state` |
|---|---|
| 参照側がbackend設定を書く必要があり、設定のtypoは`plan`まで気付けない | `source`にディレクトリを指定するだけ。パスの間違いは`carina validate`で検出できる |
| `data.terraform_remote_state.network.outputs.vpc_id`と長い | `let`で束縛して`network.vpc_id` |
| `outputs`の中身はstateを読むまで分からず補完が効かない | `exports`は型付きで補完が効く |
| typoや存在しないoutput参照は`plan`/`apply`まで気付けない | 未定義のexport参照や型不一致は`carina validate`で検出できる |
| 参照元が未applyだとplanすら失敗する | `(known after upstream apply: ...)`として未確定値で表示され、planは通る |
