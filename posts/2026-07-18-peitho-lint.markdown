---
title: Peithoにスライドのはみ出し検査（peitho lint）を入れた
date: 2026-07-18 17:44:35 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の話の続き。今回は`peitho lint`というサブコマンドを追加した話。スライドの中身がスライドの枠からはみ出していないかを検査する機能。

---

## peitho lintの仕組み

内部でやっていることは、ざっくりこんな流れ。

1. `peitho build`と同じパイプライン（parse → map → check → render）でデッキをビルド
2. 検査用のHTMLを一時ディレクトリに書き出す
3. ヘッドレスChromeでそのHTMLを開いて、各スライドのサイズを測定
4. はみ出しがあれば警告を出して、exit code 1で終了

ポイントは、実際にブラウザでレンダリングして測るところ。フォントの違いや複雑なCSSレイアウトによるずれも、ブラウザが実際に描画した結果をそのまま見るので、正確に検出できる。

---

## 検査用HTML

検査用には専用のHTML（`lint.html`）を生成している。PDFエクスポートの`pdf.html`と似た構造で、全スライドのHTMLをインラインで持った静的な1枚もの。ただしPDFとは違って`overflow: hidden`やページ区切りのラッパーを付けていない。はみ出しを検出したいので、はみ出しを隠す要素があっては困る。

各スライドには`transform: scale(1)`というidentity transformをかけている。見た目にも計算にも何も影響しないけど、これがないと絶対配置（`position: absolute`等）された子要素が自分のスライドではなくページ全体を基準に配置されてしまい、2枚目以降のスライドで偽のはみ出しを検出してしまう。実際の発表画面やPDFでもスライドにtransformがかかっているので、同じ条件にしている。

---

## 実際にはみ出しを見つけた

`peitho lint`を入れてすぐ、リポジトリの`examples/peitho-tour`デッキ（Peitho自体を紹介するデッキ）で走らせたら、こうなった。

```
warning: slide 16 content overflows the slide box vertically by 50px (content 770px, box 720px)
  help: shrink or split the slide content, or adjust the layout CSS
warning: slide 18 content overflows the slide box vertically by 104px (content 824px, box 720px)
  help: shrink or split the slide content, or adjust the layout CSS
checked 20 slide(s): 2 overflow warning(s)
```

スクリーンショットを大きく載せているスライドが2枚、垂直方向にはみ出していた。CSSの問題でスクリーンショット画像が縮小されずに元のサイズのまま表示されていたのが原因。

修正前のスライド16。タイトルの下にスクリーンショットと本文テキストが入るはずだが、スクリーンショットが大きすぎて本文テキストが表示されず、下もはみ出している。

![修正前のスライド16](/images/2026/07/peitho-lint-before-preview.png)

修正後。スクリーンショットが縮小されて枠内に収まり、本文テキストも表示されている。

![修正後のスライド16](/images/2026/07/peitho-lint-after-preview.png)

CSSを1箇所直して解消（[PR #295](https://github.com/mizzy/peitho/pull/295)）。

```
checked 23 slide(s): no overflow
```

このデッキはAIに生成させていてちゃんとチェックしていなかったので気づいていなかった。こういうケースこそ自動検査が効く。

---

## ビルドパイプラインの検査との棲み分け

Peithoには`peitho lint`以外にも、`peitho build`で走る検査がいくつかある。

- **スロット契約の検査**: スロットの型やarityの不一致をビルドエラーにする。例えばコードブロックが0〜1個のスロットに2個書くとエラー
- **未割り当てコンテンツの検出**: Markdownに書いた要素がどのスロットにも割り当てられなければビルドエラー
- **生HTML禁止**: HTMLコメント（ページ設定やスピーカーノート）以外のHTMLがMarkdownに書かれていたらビルドエラー。コンテンツとデザインの分離を徹底するため
- **未知のコード言語**: シンタックスハイライトで未知の言語タグが指定されていたらビルドエラー
- **重複スライドキー**: 同じキーが複数のスライドに振られていたらビルドエラー
- **CSS検査**: レイアウトのルートクラスが`.peitho-slide`と異なる`width`/`height`を指定していないか、`[data-slide-key=...]`セレクタが存在しないキーを参照していないか、など
- **セクション時間の整合性検査**: セクションごとの`time`の合計がfrontmatterの`time`と一致しなければビルドエラー
- **画像の参照検査**: 参照している画像が存在するかの検査

これらは構造的な整合性の検査で、HTMLをレンダリングしなくてもソースコードの解析だけで検出できる。一方`peitho lint`は、実際にレンダリングしないとわからない物理的なはみ出しを検出する。検査のレイヤーが違うので、別サブコマンドにしている。ビルドの検査はビルド時に必ず走るけど、lintはChromeが必要なので明示的に実行する形。

---

参考:

- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [前回の記事: Peithoにスマホからスライド操作できる機能を入れた](/blog/2026/07/18/1/)
