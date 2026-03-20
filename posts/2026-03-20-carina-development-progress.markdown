---
title: Carinaの開発状況（2月下旬〜3月中旬）
date: 2026-03-20 17:56:56 +0900
---

[前回の記事](https://mizzy.org/blog/2026/02/21/1/)ではClaude Codeによるイシュー駆動開発について書いた。あの時点では90個のイシューが登録され67個がクローズされた、というところだったが、あれから約1ヶ月、さらに約280個のイシューがクローズされ、約320個のPRがマージされた。

この1ヶ月でやったことを振り返っておく。

---

## AWS providerの本格整備

aws providerは以前からあったが、スキーマの手書き部分が多く、テストも不十分だった。この1ヶ月で本格的に整備した。

まず、[Smithyモデル](https://github.com/aws/api-models-aws)からスキーマを自動生成する仕組みを入れた（[#461](https://github.com/carina-rs/carina/issues/461)）。awscc providerのCloudFormationスキーマからの自動生成と同じ考え方で、AWS SDKの元になっているSmithyモデルから、リソーススキーマとread関数を自動生成する。手書きのスキーマは信用できないので、できるだけ自動生成に寄せていきたい。

---

## Acceptance testの拡充

AWS providerとAWSCC providerの両方で、acceptance testを大幅に増やした。前回の記事時点ではAWSCC providerの39ファイルだけだったが、AWS providerのテスト追加とシナリオの拡充で172ファイルになった（AWSCC: 120、AWS: 52）。基本的なCRUDに加えて、タグの削除、in-place update、create-before-destroy、属性の削除といったシナリオを整備している。

特に大きかったのは、in-place updateのテストの追加。リソースを作成した後、属性を変更してもう一度applyし、replaceではなくin-place updateが正しく行われることを確認するテスト。これを入れたことで、AWSCC providerのupdate処理にあったバグが大量に見つかった。

- Cloud Control APIのupdate用patch documentにread-onlyプロパティが含まれていた（[#807](https://github.com/carina-rs/carina/issues/807)）
- create-onlyプロパティもpatchに含まれていた（[#810](https://github.com/carina-rs/carina/issues/810)）
- JSON Patchのreplaceではなくaddオペレーションを使うべきだった（[#795](https://github.com/carina-rs/carina/issues/795)）
- 変更のない属性もpatchに含まれていた（[#737](https://github.com/carina-rs/carina/issues/737)）
- 属性の削除がpatchに反映されていなかった（[#736](https://github.com/carina-rs/carina/issues/736)）

multi-stepテスト（create → update → destroy）のインフラも整備した。テストスクリプトのシグナルハンドリング（[#793](https://github.com/carina-rs/carina/issues/793)）、destroy失敗時の検出（[#855](https://github.com/carina-rs/carina/issues/855)）、orphanedリソースの防止（[#812](https://github.com/carina-rs/carina/issues/812)）など、テストの信頼性を上げるための改善も多い。

---

## Anonymous resourceの識別と置き換え

Carinaでは、Terraformと異なり、リソースに名前をつけなくてもよい。

```crn
# Terraformだと aws_vpc.main のように名前が必要だが、Carinaでは不要
awscc.ec2_vpc {
  cidr_block = "10.0.0.0/16"
}
```

ただし、内部的には`.crn`ファイルに書かれたリソースとstateに保存されたリソースを突き合わせるためのidentifierが必要になる。Terraformは人間がつけた名前（`aws_vpc.main`の`main`の部分）をidentifierとして使うが、Carinaのanonymous resourceにはそれがない。

そこで、スキーマのcreate-onlyプロパティ（作成後に変更できない属性）の値からハッシュを計算してidentifierとしている。ただ、create-onlyプロパティが変更された場合、ハッシュが変わってidentifierも変わる。するとdifferは、`.crn`ファイルにある「新しいidentifierのリソース」とstateにある「古いidentifierのリソース」を同じリソースだと認識できず、「新しいリソースの作成」と「古いリソースの削除」という無関係な2つの操作として扱ってしまう。本来は「同じリソースの置き換え（Replace）」として認識させる必要がある。

そこで、`reconcile_anonymous_identifiers()`という仕組みを入れた（[#540](https://github.com/carina-rs/carina/issues/540)）。create-onlyプロパティの一部が一致し一部が異なる場合、同じリソースの変更であると判定し、stateにある旧identifierを引き継ぐことでReplaceとして扱う。

さらに、create-onlyプロパティを持たないリソース（EC2 EIPなど）に対しては、全属性の[SimHash](https://en.wikipedia.org/wiki/SimHash)（locality-sensitive hash）をidentifierとして使うようにした（[#592](https://github.com/carina-rs/carina/issues/592)）。通常のハッシュは1ビットでも入力が変わると出力が大きく変わるが、SimHashは入力の類似度を保存する。属性を1つ変更しても数ビットしか変わらないので、Hamming距離で同一リソースかどうかを判定できる。ただし、tagsのようなMap属性をそのまま1つのfeature（ハッシュの入力単位）としてハッシュすると、tag1つの変更でも相対的な変化が大きくなりすぎて閾値を超えてしまう問題があった。Map/List値を個々のエントリに展開（flatten）してそれぞれ別のfeatureとすることで修正した（[#885](https://github.com/carina-rs/carina/issues/885)）。

Terraformにはanonymous resourceの概念がないので、Carina独自の設計判断が多く、バグも出やすい領域だった。

---

## Create-before-destroyの名前衝突回避とカスケード更新

リソースのReplace（置き換え）を実行するcreate-before-destroyでも、Terraformにはない仕組みを入れた。

Terraformではcreate-before-destroyの際、名前の衝突は人間が`name_prefix`等で回避する必要がある。Carinaでは、リソースの`name_attribute`（S3なら`bucket_name`、IAMなら`role_name`）をスキーマから自動判定し、replaceの際に一時名を自動生成する（[#405](https://github.com/carina-rs/carina/issues/405)）。名前がupdatable（create-onlyでない）なら、旧リソース削除後に本来の名前にリネームする。create-onlyな場合はリネームできないので一時名のまま残る。

依存リソースのカスケードアップデートも入れた（[#404](https://github.com/carina-rs/carina/issues/404)）。create-before-destroyでは「新リソース作成→依存リソース更新→旧リソース削除」という順序で実行する必要があるが、この中間ステップの依存リソース更新をdifferが依存グラフから検出してplanに含める。

---

## LSPの強化

エディタ上での開発体験を良くするために、LSP（Language Server Protocol）まわりを大幅に改善した。

- プロバイダーブロックの補完（[#899](https://github.com/carina-rs/carina/issues/899)）
- リソース参照の型ベース補完（[#914](https://github.com/carina-rs/carina/issues/914)） — `route_table_id`に対してRouteTableを返すリソースだけを候補に出す
- read-only属性をリソースブロック内の補完候補から除外（[#921](https://github.com/carina-rs/carina/issues/921)）
- 属性ホバーで間違った説明を表示していた問題の修正（[#924](https://github.com/carina-rs/carina/issues/924)）
- 未定義参照の検出強化（[#871](https://github.com/carina-rs/carina/issues/871)）
- 重複letバインディングの検出（[#916](https://github.com/carina-rs/carina/issues/916)）
- 未使用letバインディングの警告（[#530](https://github.com/carina-rs/carina/issues/530)）
- ネストされたstructブロックの任意深さでの補完・診断（[#400](https://github.com/carina-rs/carina/issues/400)、[#402](https://github.com/carina-rs/carina/issues/402)）
- 非ASCII文字によるbyte/charインデックスのずれの修正（[#666](https://github.com/carina-rs/carina/issues/666)）

---

## コード生成の強化

CloudFormationスキーマとSmithyモデルからのコード生成器（codegen）に、バリデーション制約のサポートを追加した。文字列長（minLength/maxLength）、配列長（minItems/maxItems）、正規表現パターン、フォーマット制約（int64、URIなど）、デフォルト値、複数制約の合成（pattern + lengthなど）をサポートしている（[#628](https://github.com/carina-rs/carina/issues/628)〜[#636](https://github.com/carina-rs/carina/issues/636)）。

これにより、`carina validate`で実際にAWSにリクエストを投げる前に、かなり細かいバリデーションができるようになった。

---

## State管理の改善

本番運用を見据えたstate管理の改善も進めた。

- state lockの改善 — TTLを60分に延長し、lock renewalとconditional writeを追加（[#857](https://github.com/carina-rs/carina/issues/857)）
- アトミックなstate書き込み — ローカルバックエンドのstate書き込みをatomicに（[#849](https://github.com/carina-rs/carina/issues/849)）
- `--lock=false`オプション — state lockのスキップ（[#893](https://github.com/carina-rs/carina/issues/893)）
- orphanedリソースの検出 — `.crn`ファイルから消えたリソースの自動検出・削除（[#853](https://github.com/carina-rs/carina/issues/853)）
- state refreshコマンド — AWSの実際の状態とstateの同期（[#395](https://github.com/carina-rs/carina/issues/395)）
- エラーパスでのlock解放保証（[#779](https://github.com/carina-rs/carina/issues/779)）
- プロバイダースコープのstate identity — マルチプロバイダー構成でのstate破損防止（[#854](https://github.com/carina-rs/carina/issues/854)）

---

## その他の新機能

- **フォーマッター** — list-of-mapsをblock構文に変換（[#911](https://github.com/carina-rs/carina/issues/911)）
- **block構文** — `List<Struct>`属性に対するブロック構文のサポートと、codegenでの自動block_name生成（[#381](https://github.com/carina-rs/carina/issues/381)、[#759](https://github.com/carina-rs/carina/issues/759)）
- **前方参照** — 宣言順序に依存しないパーサー（[#876](https://github.com/carina-rs/carina/issues/876)）
- **循環依存検出** — リソース間の循環依存をエラーとして報告（[#665](https://github.com/carina-rs/carina/issues/665)）
- **型認識diff** — differが型情報を持って意味的な比較を行うように（[#613](https://github.com/carina-rs/carina/issues/613)）
- **ordered/unordered list** — AWSCCスキーマのリストに順序付き/順序なしのセマンティクスを追加（[#873](https://github.com/carina-rs/carina/issues/873)）

---

## リファクタリング

コードベースが大きくなってきたので、構造的な整理も進めた。main.rs（4000行）、awscc provider.rs（3500行）、differ.rs、diagnostics.rs、completion.rsなどの大きなファイルを分割し、carina-coreからvalidation、resolver、deps、config_loaderなどのモジュールを抽出した（[#382](https://github.com/carina-rs/carina/issues/382)〜[#392](https://github.com/carina-rs/carina/issues/392)、[#673](https://github.com/carina-rs/carina/issues/673)、[#787](https://github.com/carina-rs/carina/issues/787)）。

また、CLIやLSPのコードに`"aws"`や`"awscc"`といったプロバイダー名が直接書かれていたのを、`ProviderFactory`トレイト経由で動的に解決するように変更した（[#364](https://github.com/carina-rs/carina/issues/364)、[#367](https://github.com/carina-rs/carina/issues/367)）。新しいプロバイダーを追加してもCLIやLSPのコードを変更する必要がなくなった。
