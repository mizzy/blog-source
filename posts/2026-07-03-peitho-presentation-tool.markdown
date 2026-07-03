---
title: プレゼンツールPeithoをつくっている
date: 2026-07-03 19:35:00 +0900
---

[前回の記事](/blog/2026/06/23/1/)でPyxis/Argus/Irisという最近つくっているOSSを紹介したけど、もうひとつ新しいものをつくりはじめた。[Peitho](https://github.com/mizzy/peitho)というプレゼンテーションツールで、Markdownをsource of truthとして、HTMLネイティブなスライドを生成し、ブラウザでの発表までやる。

ビルドするとどんなスライドになるのかは、デモサイト[peitho.gosu.ke](https://peitho.gosu.ke/)で見られる。リポジトリの`examples/`を`peitho build`してそのまま配信したもの。発表のほうは手元でChromeで操作するので、これは後述する。

![Peithoのデモサイト](/images/2026/07/peitho-demo-landing.png)

---

## なぜつくったか

スライドをMarkdownで書く、というのはずっとやっていて、[Deckset](https://www.deckset.com/)から[k1LoW/deck](https://github.com/k1LoW/deck)へとたどってきた。Decksetはデザインの自由度がなくて、「ここがもっとこうなると見やすくなるのに」というところが変更できない、かゆいところに手が届かない感が不満だった。

k1LoW/deckは「コンテンツとデザインの分離」という思想が最高にかっこいい。Markdownで書いたコンテンツをGoogle Slidesに反映するツールで、内容はgitで管理しつつ、デザインはGoogle Slidesに任せる、という分担がきれいに決まっている。

k1LoW/deckは最高なんだけど、Google Slidesでデザインを手動調整するのは、かゆいところに手が届く反面、面倒さもともなう。これがPeithoをつくりはじめた動機。今なら、ここは生成AIに任せたい。デザイン成果物がHTML/CSSというただのコードなら、AIに書かせることができるし、gitで管理してdiffを取ったりレビューしたりもできる。

なので、k1LoW/deckの思想は受け継ぎつつ、デザインの側をGoogle SlidesではなくHTML/CSSで持つ、という形にした。Peithoの設計は三本柱にしている。

1. **コンテンツとデザインの分離** — コンテンツはMarkdown、デザインはレイアウトHTMLとCSS。両者を混ぜない（k1LoW/deckリスペクト）
2. **git管理可能なHTML/CSSレイアウト** — デザイン成果物はただのHTML/CSS。diffが取れ、レビューできる
3. **型検査されるスロット契約** — レイアウトはスロット（タイトル・本文・コードなど、コンテンツの挿し込み口）を持ち、そこに何をいくつ書けるかを宣言する。Markdownの内容がその宣言に合わないときは、黙って捨てられる代わりに行番号とヒント付きのビルドエラーになる

もうひとつ、プレゼンツールも自分が使いやすいものがほしくなった、という動機もある。ただ、まだどういう仕様にするかは詰め切れていないので、これからブラッシュアップしていく。

---

## デッキの書き方

デッキは素のMarkdownで書ける。`---`で区切ったかたまりが1枚のスライドになり、最も浅い見出しはタイトル、コードブロックはcodeスロット、残りはbodyスロットへ、という規約で自動的に割り当てられる。

````markdown
# タイトル

本文の段落。

- リストも
- 使える

---

# 次のスライド

```rust
enum Phase { Parsed, Mapped, Checked, Rendered }
```
````

`peitho build deck.md`で`dist/`に配布物（スライドHTML断片+manifest.json+index.html+CSS）が生成される。manifest.jsonはデッキの目次にあたるファイルで、スライドの順序やキー、各HTML断片のパスを持っていて、index.htmlはこれを読んでスライドを順に表示する。レイアウトとテーマはバイナリに内蔵されたデフォルトがあるので、デッキファイルがひとつあればどこでも動く。差し替えたいときはデッキファイルと同じディレクトリに`layouts/`と`css/`を置けば自動検出される、というゼロコンフィグ規約にしている。

---

## レイアウト自身がスキーマ

デザインの単位はレイアウトで、ただのHTMLファイル。Web Componentsの`<slot>`を借用した記法で、スロットの名前・受け入れるコンテンツ型・個数（arity）を宣言する。

```html
<section class="peitho-slide">
  <h1><slot name="title" accepts="inline" arity="1"></slot></h1>
  <div class="body">
    <slot name="body" accepts="blocks" arity="0..*"></slot>
  </div>
  <figure class="code">
    <slot name="code" accepts="code" arity="0..1"></slot>
  </figure>
</section>
```

これはPeithoに内蔵されているデフォルトレイアウトそのもの。ポイントは、このHTML自身がスキーマを兼ねているところ。ビルド時にPeithoがこのレイアウトをパースして、「titleはインライン要素がちょうど1個、bodyはブロック要素がいくつでも、codeはコードブロックが0〜1個」という契約を取り出し、Markdownで書いた内容がそれを満たしているかを検査する。

たとえばcodeスロットは`arity="0..1"`なので、このレイアウトのスライドにコードブロックを2つ書くとビルドが止まる。

```
error: slide 2 ('code-slide'), line 7: slot 'code' got 2 item(s), but layout 'title-body-code' allows 0..1
  = help: use a layout with more code capacity or remove one code block
```

Rustコンパイラ風に、行番号とhelpつきでエラーになる。`('code-slide')`は該当スライドに振ったキーで、これは次で説明する。

---

## キー付きper-slide調整

「このスライドだけコードの背景色を変えたい」みたいな個別調整は、まずMarkdown側で、スライドの先頭にページ設定コメントを書いて安定キーを振る。

```markdown
<!-- {"key":"payoff"} -->
# The compiler reviews every caller
```

キーを振ったスライドは、生成されるHTMLの要素に`data-slide-key="payoff"`という属性が付く。CSS側では、この属性を目印にして、そのスライドだけにスタイルを当てる。

```css
[data-slide-key="payoff"] .slot-code { background: #0d2822; }
```

このCSSも`peitho build`の検査対象になっていて、デッキに存在しないキーを指定しているとビルド時にエラーになる。スライドを消したり書き換えたりしてキーが無くなったのに、それを指定したCSSが参照切れのまま静かに残り続ける、ということもない。

---

## 複数レイアウトと型駆動ディスパッチ

レイアウトは複数持てて、どのスライドにどのレイアウトが適用されるかは次の順で決まる。

1. ページ設定コメント`<!-- {"layout":"cover"} -->`による明示指定
2. レイアウトが1枚だけなら無条件にそれ
3. 複数枚なら、スライドの内容の形（タイトルだけ・本文あり・コードあり等）にスロット契約が一致するレイアウトへ自動で振り分ける

3の型駆動ディスパッチは、ちょうど1枚に一致することが条件で、複数一致（曖昧）も0枚一致も黙って解決せずビルドエラーにして明示指定を促す。ここもk1LoW/deckのページ設定の仕組みを参考にしつつ、「黙って何かが起きる」経路を残さないようにした。

---

## peitho present

ビルドだけでなく発表もカバーしていて、`peitho present deck.md`でローカルサーバを立ててブラウザを起動する。外部ディスプレイがあればそちらにスライドをフルスクリーン、手元のディスプレイに発表者ビュー（現在/次スライド・ノート・タイマー）を自動配置する。Escで全ウィンドウとサーバがまとめて終了する。Keynote的な発表体験をブラウザでやる、というもの。

実行するとこんな感じ。1画面のMacに仮想ディスプレイを足して、それを外部ディスプレイに見立てた環境での録画。スライドが仮想ディスプレイ側にフルスクリーンで出て、手元に発表者ビューが開き、Escでまとめて終了する。

<video src="/images/2026/07/peitho-present.mp4" controls muted playsinline style="max-width:100%"></video>

---

## サンプル

Peithoのリポジトリの`examples/`に、内容もレイアウト構造もテーマも異なるサンプルを置いていて、[デモサイト](https://peitho.gosu.ke/)でそのまま見られる。

*Keynote* — クリーム地+セリフ体+中央寄せ。cover（タイトルのみ）とstatement（タイトル+本文）の2レイアウト構成で、型駆動ディスパッチが内容の形からレイアウトを振り分ける。

![Keynoteサンプル](/images/2026/07/peitho-keynote.png)

*Code Walkthrough* — ターミナル風2カラムのコード解説。codeスロットが`arity="1"`なので、毎スライドにコードが必須。シンタックスハイライトは[syntect](https://github.com/trishume/syntect)（ArgusやIrisと同じ）でビルド時にかけていて、実行時JSは無く、色はテーマCSSで定義する。

![Code Walkthroughサンプル](/images/2026/07/peitho-code-walkthrough.png)

*Lightning Talk* — ダーク+大型タイポのLT向けデザイン。レイアウトにcodeスロットが無いので、コードを書くとビルドエラーになる。

![Lightning Talkサンプル](/images/2026/07/peitho-lightning-talk.png)

この「コードを書くとビルドエラーになるレイアウト」というのが、レイアウトがスキーマを兼ねることの面白いところだと思っている。レイアウトが「このスライドに何を書けるか」を決めて、それをビルドが守らせる。

---

## 実装

ビルドコアはRust。デッキを表す型が、パース済み（`Parsed`）→スロット割り当て済み（`Mapped`）→検査済み（`Checked`）→レンダリング済み（`Rendered`）と、パイプラインの段階ごとに分かれていて、レンダラは検査済みの型しか受け取れない。つまり、検査をすっ飛ばしてレンダリングするようなコードは、実行時エラーになるのではなく、そもそもコンパイルが通らない。いわゆるtypestateパターン。Rustのdoctestには「このコードはコンパイルに失敗するはず」を検証できる`compile_fail`という仕組みがあるので、検査前のデッキをレンダラに渡すコードが実際にコンパイルエラーになること自体もテストで固定している。

一方、`peitho present`の発表時にブラウザ側で動く部分（スライド表示、キー操作、タイマー、画面間の同期）はTypeScriptで書いている。RustとTypeScriptの両方で、ビルドが出力するmanifest等の同じデータ構造を扱うことになるので、型定義はRust側にだけ置いて、TypeScript側の型は[ts-rs](https://github.com/Aleph-Alpha/ts-rs)で自動生成している。手書きの型が2つあるとズレていくので、生成した型がRust側と一致しているかをCIで検査する。

---

## 名前の由来

Peitho（ペイトー）は、力ではなく言葉で人の心を動かすことを司るギリシャ神話の女神から。プレゼンテーションはまさに言葉で人の心を動かす行為なので、ぴったりだと思ってつけた。Carina、Pyxis、Argus、Irisと続いてきた神話シリーズの並びでもある。

---

## 今後

- 実際の発表スライドを公開するサイト（スライドのMarkdownをリポジトリにpushしたら、CIがビルド=契約検査して、そのままサイトで公開されるところまで自動化したい）
- スピーカーノートのMarkdown記法（発表者ビューの表示側はできていて、記法だけ未決）
- 2カラムなど規約で曖昧になるレイアウト向けの、fenced divによる明示スロット記法
- プレゼンツールとしての機能充実

などなど。

ちなみに、プレゼンツールつくってるけど、次のプレゼンの予定は今のところ特にない。

---

参考:

- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [k1LoW/deck](https://github.com/k1LoW/deck)
- [k1LoW/deckの紹介とチュートリアル](https://www.docswell.com/s/Songmu/K137GV-about-deck)
