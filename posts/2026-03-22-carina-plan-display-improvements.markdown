---
title: Carinaのplan表示を改善した
date: 2026-03-22 19:21:17 +0900
---

Carinaの`carina plan`コマンドの表示を最近まとめて改善したので、その内容を書いておく。

---

## スナップショットテストの導入

plan表示の改善を始める前に、まずテストの仕組みを整えた。plan表示は目視で確認するのが面倒だし、変更のたびに既存の表示が壊れていないか確認するのも大変なので、[insta](https://insta.rs/)を使ったスナップショットテストを導入した。

仕組みはシンプルで、テスト用の`.crn`ファイル（とオプションでstateファイル）をfixtureとして用意し、それに対して`format_plan()`を実行した結果をスナップショットとして保存する。テスト実行時にスナップショットと一致しなければ失敗する。表示を変更した場合は`cargo insta review`で差分を確認してスナップショットを更新すれば良い。

plan表示はANSIカラーコードを含むので、スナップショットの可読性のためにカラーコードをstripしてからスナップショットに保存している。

これにより、表示の改善を安心してガンガン進められるようになった。実際のスナップショットは[こちら](https://github.com/carina-rs/carina/tree/main/carina-cli/src/snapshots)にあるので、どんな表示になるのかはここを見るとわかる。

---

## 依存関係のツリー表示

Terraformの`terraform plan`はリソースをフラットなリストで表示する。リソース間に依存関係があっても、出力上はすべて同じ階層にフラットに並ぶ。さらにリソース名のアルファベット順でソートされるので、依存関係のあるリソース同士が離れた位置に表示されることもある。結果として、どのリソースがどのリソースに依存しているかは人間が属性値を見て判断する必要がある。

Carinaではリソース間の依存関係をツリー構造で表示するようにした。

![carina plan のツリー表示](/images/2026/03/carina-plan-tree.png)

VPCの下にroute_tableとsubnetがぶら下がっているのが視覚的にわかる。Create、Update、Destroyが混在する場合でも、依存関係の中で何が起きるかがひと目でわかる。

![Create、Update、Destroyが混在するplan表示](/images/2026/03/carina-plan-mixed.png)

---

## read-only属性の表示

リソースを作成する際、ARNやIDのようなread-only属性はAWS側が自動的に割り当てる。以前のplan表示ではこれらは完全に省略されていたが、Terraformと同様に`(known after apply)`として表示するようにした。

![read-only属性の表示](/images/2026/03/carina-plan-read-only.png)

これにより、リソース作成後にどんな属性が生えてくるのかがplanの時点でわかるようになった。

---

## デフォルト値の表示

スキーマにデフォルト値が定義されている属性も、ユーザが明示的に指定していなければplanに表示されていなかった。これを`# default`アノテーション付きで表示するようにした。

![デフォルト値の表示](/images/2026/03/carina-plan-default-values.png)

ユーザが指定した属性、デフォルト値が適用される属性、apply後に確定するread-only属性、という3層が一目でわかる。

---

## 未変更属性のサマリ表示

Update/Replaceの際、変更されない属性がいくつあるかを`# (3 unchanged attributes hidden)`のようなサマリ行で表示するようにした。

![未変更属性のサマリ表示](/images/2026/03/carina-plan-map-diff.png)

---

## `--detail`フラグによる表示レベルの制御

`--detail`フラグで表示レベルを制御できるようにした。3つのモードがある。

- **full**（デフォルト）: すべての属性を表示。未変更・デフォルト・read-onlyも含む
- **explicit**: ユーザが`.crn`ファイルで明示的に指定した属性のみ表示
- **none**: リソース名と依存ツリーのみ表示

デフォルトの`--detail full`だとこう。

![--detail full の表示](/images/2026/03/carina-plan-detail-full.png)

`--detail explicit`だとこうなる。

![--detail explicit の表示](/images/2026/03/carina-plan-detail-explicit.png)

`--detail none`だとリソース名と依存ツリーのみになる。リソース数が多いときに全体の構造だけさくっと確認したい場合に便利。

![--detail none の表示](/images/2026/03/carina-plan-detail-none.png)

---

## TUI（Terminal UI）の追加

CLIの表示とは別に、`--tui`フラグでTUI（Terminal UI）も使えるようにした。ratatuiとcrosstermで実装している。

<video src="/images/2026/03/carina-tui.mp4" controls muted playsinline style="max-width:100%"></video>

主な機能は以下の通り。

- **ツリービュー + 詳細パネル**: 画面の70%がリソースの依存関係ツリー、30%が選択したリソースの属性詳細。Tabキーでパネル間を切り替える
- **検索・フィルタ**: `/`キーで検索モードに入り、リソース名やタイプでフィルタ。Tabキーで補完、`n`/`N`でマッチ間を移動
- **リソース参照ナビゲーション**: 詳細パネルで`vpc_id: vpc.vpc_id`のようなリソース参照をEnterキーで辿れる。Backspaceで戻る
- **色分け**: Create（緑）、Update（黄）、Replace（マゼンタ）、Delete（赤）で操作の種類が一目でわかる

---

これらの改善は、Terraformのplan表示を参考にしつつ、ツリー表示やTUIのようなCarina独自の機能も加えている。applyする前にリソースの全体像を把握しやすくなった。
