---
title: Claude CodeによるCarinaのイシュー駆動開発
date: 2026-02-21 17:11:50 +0900
---

[Carina](https://github.com/carina-rs/carina)の開発はこれまでコード自体をほとんど見ずにやらせていたので、コード全体の品質や一貫性がどうなっているかは把握していなかった。

そこで、Claude Codeに問題点をGitHubイシューとして登録させ、そのイシューをひとつずつ潰させる、というサイクルを回してみた。イシューの対応はすべて自分でPRをレビューしてからマージしていて、これは自分自身のコードや設計の理解を深める意味もある。10日間回した結果、90個のイシューが登録され、67個がクローズされた。

---

## 作業フロー

具体的な作業フローはこんな感じ。

1. `/pick-issue`スキルを呼ぶと、一番新しいオープンイシューをpickしてくる
2. Claude Codeがイシューを修正してPRを出す
3. `/self-review`スキルで5回セルフレビューさせる
4. セルフレビューの過程でPRに関連する問題が見つかればその場で修正、無関係な問題であれば新たにイシューを登録
5. 自分がPRをレビューしてマージ
6. 1に戻る

イシューの優先順位付けは特にやっていない。`/pick-issue`が一番新しいイシューを拾ってくるので、基本的にはFILO（後入れ先出し）で回している。気になるイシューがあれば`/pick-issue 123`のように番号指定することもあるが、大半はそのまま最新のものを拾わせている。修正の過程で見つかった新しいイシューが次にpickされやすいので、関連する問題が芋づる式に片付いていく、という副次的な効果もある。

イシューの登録元はセルフレビューだけではなく、コードベース全体のレビューや、acceptance testの実行結果から問題を拾ってイシューに登録させる、ということもやっている。

このフローだと、完全にボトルネックは人間によるレビュー。レビュー対象はPRのコードだけでなく、Claude Codeが立てたplanも含まれる。plan段階でプロジェクトの方針とまったく逆方向の提案をしてくることもあるので、方針がおかしければそこで差し戻す。

---

## Day 1: 23個のイシュー

まずClaude Codeにコードベース全体のレビューとacceptance testの実行をさせたところ、1日で23個のイシューが登録された。バグ、設計上の問題、不足している機能など、色々なカテゴリのイシューが一気に出てきた。たとえば:

- `Effect::Delete`にリソース識別子が含まれていない（[#135](https://github.com/carina-rs/carina/issues/135)）
- IGWのデタッチエラーが握りつぶされている（[#136](https://github.com/carina-rs/carina/issues/136)）
- `create_plan()`が削除すべきリソースを検出しない（[#137](https://github.com/carina-rs/carina/issues/137)）
- Structバリデーションがunknown fieldを無視している（[#138](https://github.com/carina-rs/carina/issues/138)）
- plan fileの保存とapplyが欲しい（[#142](https://github.com/carina-rs/carina/issues/142)）
- `name`属性をリソース識別子として使うのをやめるべき（[#148](https://github.com/carina-rs/carina/issues/148)）
- S3バケットの中身が空でないと削除できない（[#160](https://github.com/carina-rs/carina/issues/160)）

---

## Day 2-3: 修正がさらなる問題を呼ぶ

初日のイシューを潰し始めると、修正の過程で新たな問題が次々と見つかった。

特に大きかったのが、`--detailed-exitcode`（[#130](https://github.com/carina-rs/carina/issues/130)）の実装と、acceptance testへのapply後plan verify追加（[#129](https://github.com/carina-rs/carina/issues/129)）。apply後にもう一度planを実行して差分がないことを確認する仕組みを入れたところ、べき等性の問題が大量に見つかった。

- enum値の大文字小文字が正規化されない（[#174](https://github.com/carina-rs/carina/issues/174)、[#176](https://github.com/carina-rs/carina/issues/176)）
- AWSから返ってきた値にスキーマにない余分なフィールドが含まれている（[#172](https://github.com/carina-rs/carina/issues/172)）
- `List<Struct>`の比較が順序依存で、false diffが出る（[#171](https://github.com/carina-rs/carina/issues/171)）

acceptance testを実際に10アカウント並列で回すと、テストランナー自体のバグも出てきた。ワーカーサブシェルがエラーで黙って停止する問題（[#170](https://github.com/carina-rs/carina/issues/170)）や、apply失敗時のリソースクリーンアップ（[#180](https://github.com/carina-rs/carina/issues/180)）など。こういった問題もすべてイシューとして登録させて、同じフローで潰していった。

---

## Day 4-6: enumの大掃除

べき等性の問題を追いかけていくと、根本原因がenum処理の一貫性のなさに行き着いた。

enumの正規化・バリデーション・変換まわりはアドホックなコードが多く、`normalize_instance_tenancy`、`normalize_region`のような個別のハードコードされた関数が各crateに散らばっていて、それぞれ微妙に違う挙動をしていた。

ここからenumの大掃除が始まった。

- 複数crateに散らばっていた`normalize_*`関数を`carina-core::utils`に集約（[#201](https://github.com/carina-rs/carina/issues/201)、[#204](https://github.com/carina-rs/carina/issues/204)）
- `get_enum_valid_values()`をスキーマ定義から自動生成（[#186](https://github.com/carina-rs/carina/issues/186)）
- 大文字小文字を区別しないenum照合（[#217](https://github.com/carina-rs/carina/issues/217)）
- `extract_enum_value`パターンの重複排除（[#203](https://github.com/carina-rs/carina/issues/203)、[#209](https://github.com/carina-rs/carina/issues/209)、[#219](https://github.com/carina-rs/carina/issues/219)）
- ハードコードされた特殊処理の削除（[#199](https://github.com/carina-rs/carina/issues/199)）

ひとつ直すと「この関数と同じパターンがあっちにもある」「この関数、もう使われてないのでは」と芋づる式にイシューが出てくる。Claude Codeに「さっき直した関数と同じパターンが他にもないか探して」と指示すると、重複を見つけてイシューを登録し、修正してくれる。

---

## Day 7: APIの引き締め

enumの大掃除が一段落したところで、今度はAPIの公開範囲の見直しに入った。

- `validate_iam_policy_document`をprivateに（[#231](https://github.com/carina-rs/carina/issues/231)）
- `validate_namespaced_enum`を`pub(crate)`に（[#233](https://github.com/carina-rs/carina/issues/233)）
- type factory関数を`pub(crate)`に（[#236](https://github.com/carina-rs/carina/issues/236)）
- `validate_*`関数のエラーメッセージ形式を統一（[#228](https://github.com/carina-rs/carina/issues/228)）

この日は13個のイシューが登録され、13個すべてクローズ。登録してすぐ直す、のリズムが回っていた。

同じ日に、コード生成まわりのバグもいくつか見つかった。VPC Endpoint IDの演算子優先順位問題（[#244](https://github.com/carina-rs/carina/issues/244)）は、初日に登録されていた[#132](https://github.com/carina-rs/carina/issues/132)の派生バグで、5日かけて全箇所を潰した形になる。

---

## Day 8-9: CLIからプロバイダ固有ロジックを追い出す

初日のレビューで「carina-coreやCLIにプロバイダ固有のロジックが入り込んでいる」（[#155](https://github.com/carina-rs/carina/issues/155)）というイシューが登録されていた。ここに本格的に着手。

`ProviderFactory`トレイトを導入して、設定バリデーション、リージョン抽出、プロバイダインスタンス化、スキーマ読み込みを抽象化した（[#259](https://github.com/carina-rs/carina/issues/259)）。CLI側にあった`"aws"`や`"awscc"`のmatchブロックがなくなり、新しいプロバイダを追加してもCLIのコードを変更する必要がなくなった。

enum処理のうちプロバイダ固有のものも、CLI側からProvider trait側に移動させた（[#185](https://github.com/carina-rs/carina/issues/185)、[#190](https://github.com/carina-rs/carina/issues/190)）。

---

## 全体を振り返って

10日間のサイクルで実装された主な機能・改善をまとめておく。

**新機能:**

- plan fileの保存とapply（`carina plan --out=plan.json` / `carina apply plan.json`）
- 孤児リソースの自動検出・削除
- `--detailed-exitcode`フラグ
- `lifecycle { force_delete }`メタ引数
- `<attr>_prefix`サポート（ランダムサフィックス付きリソース名生成）
- unknown fieldのLevenshtein距離によるサジェスト
- enumエイリアス（`IpProtocol`の`all` → `-1`など）

**リファクタリング:**

- リソース識別子の再設計（`name`属性からの分離）
- enumシステムの全面書き直し（17 PR）
- 型の厳密化（String → 具体的なARN型、リソースID型、enum型など）
- `ProviderFactory` traitによるCLI/プロバイダの分離
- API公開範囲の引き締め

**テスト:**

- テスト数: 250 → 454
- apply後のplan verifyによるべき等性チェック
- テストランナーのシグナルハンドリング、事前認証、クリーンアップ改善
