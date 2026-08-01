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

CSSを解析する方向は最初から捨てた。カスケードも継承も解決できないし、`em`や`%`や`clamp()`のような相対値は最終的に何pxになるか分からない。知りたいのは「実際に何ptで表示されるのか」なので、レンダリングした結果を測るのが素直だと思う。

---

## スロット単位のはみ出し検査

もうひとつは、既存のはみ出し検査に空いていた穴を塞ぐもの。既存のものはスライドの枠からはみ出したかを見る検査で、今回足したのはその内側、スロットからはみ出したかを見る検査になる。

Peithoのスロットのコンテナには`overflow: hidden`が効いている。すると、あふれた中身は描画されないので、要素の大きさをブラウザに問い合わせても、あふれたぶんまで含んだ大きさは返ってこない。既存の検査はスライドの枠を基準に測っていたので、スロットの中で静かに切れているぶんには何も検出できていなかった。

どのくらい検出できていなかったかというと、こう。12項目の箇条書きを1枚に置いただけのデッキ（デフォルトのテーマとレイアウト）で、

![12項目の箇条書きで最後の項目が下端で切れているスライド](/images/2026/08/peitho-lint-slot-overflow.png)

12個目が半分に切れている。この状態で、以前の`peitho lint`は「はみ出しなし」と言っていた。スライドの枠自体は超えていないので、当時の測り方では何も起きない。

今は、要素ごとに「中身の本来の大きさ」（`scrollHeight`/`scrollWidth`）と「見えている領域の大きさ」（`clientHeight`/`clientWidth`）を比べて、実際にクリップが起きている要素を探している。差が出ていれば、そのぶんが隠れている。同じデッキがこう出る。

```
warning: slide 1 content overflows the `body` slot vertically by 17px
  help: shrink or split the slide content, or adjust the layout CSS
```

### 警告であってエラーではない

これはビルドエラーにはしていない。Peithoは「黙って何かが起きる」経路を潰す方針で、パーサが未知の構造を飲み込むとか、スロット契約の違反を無視するとかはビルドエラーにしている。ただしそれらはPeitho自身が決められる、答えの決まっている話。

スロットのはみ出しはブラウザのレンダリング結果で、フォントやテーマCSSや環境で変わるし、意図的なこともある（長いログを枠に収めて見せる、とか）。判断が要るものは警告、というのが既存のはみ出し検査や文字サイズ検査と揃った扱いになる。

実際、これを入れたらPeitho自身の紹介に使っているサンプルデッキで、コードブロックが14px切れているスライドが見つかった。CSSを調整して直した。

---

## 報告されるpx数は多めに出る

「17px切れている」と報告していても、実際に見えなくなっているテキストがちょうど17px分あるとは限らない。多めに出る。

はみ出した量として測っているものに、最後の段落の下マージンが混ざるため。マージンは何も描画しないので、そのぶんは「切れているテキスト」ではない。手元で試した例だと、47pxと報告されたうち、本当に見えなくなっていたのは24px分だった。

極端な話、テキストは全部収まっていてマージンだけが枠を越えている場合でも警告が出る。実際に描画されているものだけを測ろうとすると機構がだいぶ増えるので、今のところは受け入れている。

---

## サンプルデッキには効きすぎる

さっき出てきた紹介用のサンプルデッキ（[Peitho Tour](https://peitho.gosu.ke/examples/peitho-tour/)）に今の`peitho lint`をかけると、はみ出し警告は0個になったけど、文字サイズ警告のほうが19個出る。

```
warning: slide 11 has text at 12pt, below the recommended 24pt: "---"
warning: slide 14 has text at 14.3pt, below the recommended 24pt: "error: slide 2 ('code-slide'), line 7: s…"
warning: slide 16 has text at 16.5pt, below the recommended 24pt: "Preview watches the deck, its layouts, a…"
```

中身は12ptから18.8ptで、大半はMarkdownの記法やCLIの出力をそのまま見せているスライド。ただこのデッキはPeithoの機能を一通り使ってみせるためのサンプルで、実際に発表するものではない。会場の後ろから読めるか、という基準を当てる対象ではなかった。

閾値を固定にした以上こうなるので、デッキの性質に応じて設定で変えられるようにするか、という話にはなりそう。

---

参考:

- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [ARL: PowerPoint Guidelines for Presenters](https://www.arl.org/accessibility-guidelines-for-powerpoint-presentations/) — 24ptの根拠にしたガイドライン
- [前回のlintの記事: Peithoにスライドのはみ出し検査（peitho lint）を入れた](/blog/2026/07/18/2/)
