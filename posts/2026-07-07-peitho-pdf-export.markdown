---
title: PeithoにPDFエクスポート機能を入れた
date: 2026-07-07 20:16:02 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の話がここのところ続いているけど、今日はPDFエクスポート機能について。`peitho export pdf deck.md -o out.pdf`でデッキをPDFに書き出せるようにした。

---

## ヘッドレスChromeに印刷させる

PDFの生成はヘッドレスChromeにやらせている。`peitho build`と同じパイプライン（parse → map → check → render）でデッキをレンダリングしたあと、PDF用のエントリHTML（pdf.html）を一時ディレクトリに書き出して、そこにChromeを`--print-to-pdf`で向ける、という流れ。

```
$CHROME --headless=new \
        --disable-gpu \
        --no-sandbox \
        --no-pdf-header-footer \
        --virtual-time-budget=10000 \
        --user-data-dir=<tmp>/chrome-profile \
        --print-to-pdf=<out> \
        file://<tmp>/pdf.html
```

PeithoのスライドはそもそもHTML/CSSでブラウザ描画前提でつくっているので、ブラウザにそのまま描画させて印刷させるのが素直だった。

---

## PDF専用のエントリHTML

PDF用には`pdf.html`という専用のエントリHTMLを生成している。通常の`peitho build`の出力と並べるとこんな構成。

```
# peitho build の出力
index.html      # ビューワのガワ。スライド本体は入っていない
manifest.json   # スライド一覧などのメタデータ
slides/         # スライド1枚ごとのHTML断片
peitho.css
assets/
fonts/

# peitho export pdf が一時ディレクトリに書き出すもの
pdf.html        # 全スライドのHTMLをインラインで抱えた静的な1枚もの
peitho.css
assets/
fonts/
```

`peitho.css`（テーマCSS）、`assets/`（画像）、`fonts/`は両者共通で、同じコードでコピーしている。違うのはエントリHTMLまわり。

buildの`index.html`は中身が空のビューワで、開くとJavaScriptが`manifest.json`と`slides/*.html`をfetchして、1枚ずつcanvasに表示し、キー操作でページを送る。fetchするのでHTTPで配信する前提のつくり。

`pdf.html`はその逆で、全スライドのHTMLを最初から埋め込んであって、fetchもmanifestも無い。なので`file://`でそのまま開けて、ローカルサーバーを立てる必要もない。

そして1スライドが1PDFページになるように、`@page`でページサイズをスライドと同じアスペクト比にして、スライドごとに`page-break-after: always`で改ページを入れている。Chromeはこれをそのまま印刷するだけ。

```html
@page {
  size: 1920px 1080px;
  margin: 0;
}
.peitho-slide-wrap {
  page-break-after: always;
}
```

---

基本的な仕組みはここまで。素直に`--print-to-pdf`を叩けば出力自体は得られるんだけど、実際に出てきたPDFを使おうとするといろいろな問題を踏んだ。ここからは、こういう問題が出たのでこういう対応をした、という話を順番に。

---

## グラデーションでPreview.appが重くなる

まず、exportしたPDFをmacOSのPreview.appで開くと、やたら重い。同じPDFをChromeで開くと何ともないので、PDF自体が壊れているわけではない。

ChromeのPDF生成はSkia（Chromeの2D描画ライブラリ）が担っている。画面に表示するときは、レイアウト計算の結果から「ここに矩形を塗る」「ここにテキストを描く」といった描画命令の列が作られて、Skiaがそれをピクセルに変換する。PDF出力のときも描画命令を作るところまでは同じで、最後だけSkiaの`SkPDF`バックエンドがピクセルではなくPDFの描画命令に変換してファイルに書き出す。

ピクセル化せずPDFの命令として出すということは、その命令をPDF viewer側が解釈することになる。CSSのgradientはPDFのShading Patternとして埋め込まれて、仕様上は正しいPDFなんだけど、Preview.appはこのShading Patternの評価が異様に遅い。

なので、印刷経路に入る直前にin-pageスクリプトでgradientを画像に事前ラスタ化して、SkPDFがShading Patternを使わない経路に落ちるようにした（[PR #144](https://github.com/mizzy/peitho/pull/144)）。

---

## box-shadowが黒い矩形になる

次に、`box-shadow`を使っているところが、Preview.appで黒い矩形になって表示される。これもChromeで開けば普通に影として描画される。

SkPDFは`box-shadow`を「黒いべた塗り + 透明度の分布を持つSoft Mask（`/S /Luminosity`）」の組み合わせとして吐く。これも仕様上は正しいんだけど、Quartz（macOSの描画エンジン）がこのSMaskの合成に失敗する。本来は縁に向かって透明になっていくぼかしとして描かれるはずの影が、ぼかしの情報を持つSMaskが効かないまま下地だけ描画された結果、不透明の黒いべた塗りの長方形として出てしまう。Luminosity SMaskはPDFの正式な仕様なので、これは仕様通りのPDFを描画できていないQuartz側のバグと言っていいと思う。とはいえQuartzを直すわけにはいかないので、PDFを吐く側で回避するしかない。

これもgradientと同じ形で、印刷時に`box-shadow`を事前ラスタ化してSMaskを使わない経路に落とした（[PR #153](https://github.com/mizzy/peitho/pull/153)）。

ここで得た学びは、同じPDFでも開くviewerによって見え方が変わる、ということ。PDFは画像のようにピクセルが固定されたものではなく、viewerが命令を解釈して描画するものなので、viewerの実装差がそのまま見た目の差になる。なのでexportしたPDFをChromeで開いて確認できても安心はできなくて、配布先で使われそうなviewer、特にPreview.appでも開いて確認する必要がある。

---

## ラスタ化がLinuxでは効いていなかった

さらに厄介だったのが、この`box-shadow`のラスタ化がmacOSでは効いているのにLinuxでは効いていない、という現象。手元のmacOSではPreview.appで黒矩形が消えているのに、CI（Linux）で生成したPDFにはまだ黒矩形が出ていた。

原因はラスタ化スクリプトの待ち方にあった。この処理は、影を画像に変換して、元の影と差し替える、というもの。画像への変換自体はLinuxでも正常に完了していて、画像データはできていた。ただ、差し替える前に「画像が描画に使える状態になったか」を`image.decode()`というAPIで確認していて、これがLinuxのヘッドレスChromeでは完了もせずエラーにもならず、黙って止まる。画像はできているのに完了報告だけが来ない状態で、エラーにならないので`catch`でも拾えず、最後の差し替えが実行されない。

そしてChromeの`--print-to-pdf`は、ページの処理が一定時間で終わらなければ、その時点の状態のまま印刷する、という挙動をする。なのでスクリプトが止まったままタイムアウトして、影がラスタ化されていないPDFが、何のエラーも出さずに吐かれていた。macOSでは`image.decode()`が正常に完了するので、この問題はLinuxでexportしたときにだけ起きる。

修正としては、ハングするのはChromium側の挙動でこちらからは直せないので、`image.decode()`に依存するのをやめて、画像のloadイベントを待つ形に変えた（[PR #157](https://github.com/mizzy/peitho/pull/157)）。loadイベントはLinuxでも正常に発火するし、印刷に使う画像の準備確認としてはこれで十分だった。あわせて、それまでLinux CIで一度も走っていなかったChrome実行E2Eをちゃんと動かすようにジョブを追加した。「無音でスキップされる」経路を残すと、こういう環境差のバグが検出されないままリリースされてしまう。

---

## exportする環境によってフォントが変わってしまう

もうひとつハマったのがフォント。手元のmacOSでexportしたPDFは日本語がちゃんと出るのに、Linux CIでexportしたPDFは中華フォント（WenQuanYi Zen Hei）や欧文フォント（DejaVu Sans）にfallbackしていて、見た目が全然違うものになる、という現象。

Chromeは`--print-to-pdf`のときにシステムに入っているフォントを埋め込む挙動なので、そもそもホストにフォントが無いとfallbackが走る。macOSにはヒラギノや游ゴシックが入っているけどLinux CIには入っていない、というただそれだけの環境差で、同じデッキから別物のPDFが出てしまう。

対処として、frontmatterに`fonts:`キーを足して、デッキと一緒にフォントファイル自体を持ち運ぶようにした（[PR #160](https://github.com/mizzy/peitho/pull/160)）。デッキと同じディレクトリに`fonts/`があれば自動で拾うし、明示的にパスを指定してもいい。ファイル単体でもディレクトリでも受ける（`@fontsource/inter/`のようなネスト構造もOK）。`peitho build`はもちろん、`peitho export pdf`の一時ディレクトリにも、発表用キャッシュにもコピーされるので、ホスト環境に依存せずサイト/PDF/発表者ツールで同じフォントで描画される。

使う側は、デッキ用のCSSに`@font-face { src: url("fonts/NotoSansJP.woff2") }`のように書くだけ。出力先では`peitho.css`と`fonts/`が同じディレクトリに並ぶので、この相対パスがそのまま解決される。

---

## 参考

- [PR #139](https://github.com/mizzy/peitho/pull/139) — PDFエクスポート本体
- [PR #144](https://github.com/mizzy/peitho/pull/144) — グラデーションのラスタ化
- [PR #153](https://github.com/mizzy/peitho/pull/153) — box-shadowのラスタ化
- [PR #157](https://github.com/mizzy/peitho/pull/157) — Linuxでの`image.decode()`ハング修正とLinux CI E2E追加
- [PR #160](https://github.com/mizzy/peitho/pull/160) — `fonts:`キーの追加でホスト環境非依存に
- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
