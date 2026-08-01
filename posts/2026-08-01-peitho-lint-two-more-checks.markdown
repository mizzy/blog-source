---
title: peitho lintに検査を2つ足した
date: 2026-08-01 23:42:38 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の話の続き。[以前入れた](/blog/2026/07/18/2/)`peitho lint`に検査を2つ足した。ひとつは文字が小さすぎないかの検査（[PR #386](https://github.com/mizzy/peitho/pull/386)）、もうひとつはスロット単位のはみ出し検査（[PR #389](https://github.com/mizzy/peitho/pull/389)）。

---

## 文字サイズの検査

24ptを下回るテキストがあるスライドに警告を出すようにした。24ptという数字は、学会や大学が出している発表のアクセシビリティガイドラインでよく見る下限を採ったもの（[ARLのガイドライン](https://www.arl.org/accessibility-guidelines-for-powerpoint-presentations/)は「All slides should use a minimum font size of 24 points.」と書いている）。とはいえ普遍的な基準というわけでもなくて、用途別に12pt以上でいいとする資料もあれば、会場が大きいなら30pt以上と言う資料もある。閾値は固定にしていて、今のところ設定で変えることはできない。必要になったらfrontmatterで指定できるようにすればいいので、まずは設定を増やさない側に倒している。

```
warning: slide 3 has text at 18pt, below the recommended 24pt: "Some long caption that was shrunk to f…"
  help: raise the font size in the layout CSS, or move content to another slide instead of shrinking it
```

既存のはみ出し検査は「枠から出ている」という見た目の破綻を捕まえるので、手元で見れば分かる。一方こっちは、手元のディスプレイでは普通に読めてしまう。会場の後ろから読めないことに気づくのは本番中、というやつ。

小さくすること自体が悪いわけではなくて、意図的に縮めている場面はいくらでもある。ただ、意図的かどうかとは無関係に「後ろの席から読めるか」は決まるので、閾値を切って機械に見てもらう価値がある。

### CSSを読むのではなくブラウザに測らせる

実装は、CSSの静的解析ではなく、既存のlintのヘッドレスChrome実行に相乗りする形にした。スライドごとにテキストノードを走査して、その親要素の計算済み`font-size`を読む。

CSSを解析する方向は最初から捨てた。カスケードも継承も解決できないし、`em`や`%`や`clamp()`のような相対値は最終的に何pxになるか分からない。知りたいのは「実際に何ptで表示されるのか」なので、レンダリングした結果を見る以外に答えようがない。

---

## スロット単位のはみ出し検査

もうひとつは、既存のはみ出し検査に空いていた穴を塞ぐもの。

Peithoのスロットのコンテナには`overflow: hidden`が効いている。すると、はみ出した中身は描画されないので、`getBoundingClientRect()`もはみ出したぶんまで伸びてくれない。既存の検査はスライドの枠を基準に測っていたので、スロットの中で静かに切れているぶんには何も検出できていなかった。

どのくらい検出できていなかったかというと、こう。12項目の箇条書きを1枚に置いただけのデッキ（デフォルトのテーマとレイアウト）で、

![12項目の箇条書きで最後の項目が下端で切れているスライド](/images/2026/08/peitho-lint-slot-overflow.png)

12個目が半分に切れている。この状態で、以前の`peitho lint`は「はみ出しなし」と言っていた。スライドの枠自体は超えていないので、当時の測り方では何も起きない。

今は各要素の`scrollHeight`/`scrollWidth`を`clientHeight`/`clientWidth`と比べて、実際にクリップが起きている要素を探す。同じデッキがこう出る。

```
warning: slide 1 content overflows the `body` slot vertically by 17px
  help: shrink or split the slide content, or adjust the layout CSS
```

### 隠れている要素をどう除外するか

素直に全要素を測ると、数式を使ったスライドが軒並み100px前後のはみ出しを報告してくる。KaTeXがアクセシビリティ用のMathMLのコピーを`clip: rect(1px,1px,1px,1px)`で1x1pxに畳んで隠しているためで、これは設計通りクリップされている。

`katex-mathml`というクラス名で除外すれば済む話ではある。実際、文字サイズ検査のほうは`.peitho-footnotes`をクラス名で除外している。ただしそっちはPeitho自身が生成する構造で、KaTeXのそれは他人のライブラリの内部マークアップ。名前が変われば黙って壊れるし、同じパターンを使う次のライブラリ（`sr-only`とか）が出てくるたびに除外を足すことになる。

除外したいものは「KaTeXのMathML」ではなく「視覚的に隠れている要素」なので、その性質を直接見る形にした。

```js
function isVisuallyHidden(el, style) {
  return el.clientWidth <= 1 || el.clientHeight <= 1
      || style.visibility === "hidden" || style.visibility === "collapse";
}
```

この述語は最初、`clip`と`clip-path`も見ていた。それを外したのは、実際に測ってみたら検出漏れのほうが大きかったから。`body`スロットで17pxのはみ出しを報告していたデッキが、`.body`に`clip-path: inset(0 round 12px)`——ただの角丸——を付けた途端に何も報告しなくなる。`clip-path`は普通に見えている要素に対する装飾として使われるものなので、これで隠れていると判定してはいけない。

一方でKaTeXのMathMLは`clientWidth: 1, clientHeight: 1, clipPath: "none"`なので、サイズだけ見れば除外できる。`clip-path`の枝は何も守っていないのに、実在する検出を潰していたことになる。外した上で、角丸のケースは回帰テストで固定した。

### 警告であってエラーではない

これはビルドエラーにはしていない。Peithoは「黙って何かが起きる」経路を潰す方針で、パーサが未知の構造を飲み込むとか、スロット契約の違反を無視するとかはビルドエラーにしている。ただしそれらはPeitho自身が決められる、答えの決まっている話。

スロットのはみ出しはブラウザのレンダリング結果で、フォントやテーマCSSや環境で変わるし、意図的なこともある（長いログを枠に収めて見せる、とか）。判断が要るものは警告、というのが既存のはみ出し検査や文字サイズ検査と揃った扱いになる。

実際、これを入れたらPeitho自身を紹介する`examples/peitho-tour`デッキのスライド11でコードブロックが14px切れているのが出てきた。エラーにしていたら「linterを直したら既存のデッキが壊れた」ということになって、デッキの修正を同じPRに巻き込むことになる。

なおこのスライド、測ってみたらコードの行自体は11行とも描画されていて、切れていたのはコードブロックの下パディングだった。最終行がブロックの枠に貼り付いて見えていたのはそのため。なのでデッキの内容を削るのではなく、このスライドに元からあったCSSのオーバーライド（コメントに「shrink it to fit」と書いてあって、14px足りていなかった）を調整して直した。

---

## 検出できる量は正確ではない

`scrollHeight - clientHeight`が測っているのはスクロール可能な領域で、Chromeではここに最後の子要素の下マージンが含まれる。マージンは何も描画しないので、報告されるpx数は実際に隠れている量より大きく出る。

8段落のデッキで測ったら、lintは47pxと言うが、実際に見えなくなっているテキストは24pxぶんだった。差の23pxは、テーマが`.slot-body p`に付けている`margin: 0 0 24px`。

この場合は内容が本当に切れているので数字がずれているだけだけど、最後の子要素は収まっていてマージンだけが枠を越える、という組み方をすると、何も隠れていないのに警告が出る。実際に作ってみたら再現した（60pxの箱に50pxの段落と40pxの下マージンで、30pxと報告される）。

これは今のところ受け入れている。描画されている内容を測ろうとすると、最後の子孫要素の`getBoundingClientRect()`をコンテナのパディングボックスと比べることになって、必要な機構がだいぶ増える。手元の18個のサンプルデッキを全部走らせた限り、この形に当たるデッキは無かった。将来「47pxと言われたけど1行しか消えていない」という報告が来たときに、新しいバグではなくこれだと分かるように書き残しておく。

---

## 自分のデッキの粗が出てくる

`peitho-tour`デッキに今の`peitho lint`をかけると、はみ出し警告は0個だけど、文字サイズ警告が19個出る。

```
warning: slide 11 has text at 12pt, below the recommended 24pt: "---"
warning: slide 14 has text at 14.3pt, below the recommended 24pt: "error: slide 2 ('code-slide'), line 7: s…"
warning: slide 16 has text at 16.5pt, below the recommended 24pt: "Preview watches the deck, its layouts, a…"
```

中身を見ると、12ptから18.8ptの範囲で、大半はコードブロックやコマンド出力を載せているスライド。Peithoの紹介デッキなので、Markdownの記法やCLIの出力を小さめのフォントで見せている箇所が多い。読める・読めないの判断は要るにせよ、24ptを下回っているのは事実なので、検査としては正しく効いている。

---

参考:

- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [ARL: PowerPoint Guidelines for Presenters](https://www.arl.org/accessibility-guidelines-for-powerpoint-presentations/) — 24ptの根拠にしたガイドライン
- [前回のlintの記事: Peithoにスライドのはみ出し検査（peitho lint）を入れた](/blog/2026/07/18/2/)
