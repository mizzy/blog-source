---
title: Peithoにデッキの言語指定（lang frontmatter）を入れた
date: 2026-07-31 19:11:44 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の話の続き。今回はデッキのfrontmatterに`lang`キーを追加して、生成されるHTMLの`<html lang>`属性を指定できるようにした話。対応するPRは[#339](https://github.com/mizzy/peitho/pull/339)。

---

## `<html lang="en">`固定だった問題

これまでPeithoが生成するHTMLは、すべて`<html lang="en">`で固定だった。日本語のデッキでも英語宣言のまま。セマンティクスとして正しくない、というだけならまだしも、実害もある。

Chromiumの`word-break: auto-phrase`は、テキストの言語に応じた解析を行い、自然な区切り（日本語なら文節）を優先して改行してくれるCSSプロパティ。解析には[BudouX](https://github.com/google/budoux)（日本語・簡体字中国語・繁体字中国語・タイ語に対応した機械学習ベースの改行位置解析ライブラリ）のC++移植が使われている（[Chrome 119の紹介記事](https://developer.chrome.com/blog/css-i18n-features)）。言語ごとの解析なので、効かせるには対象のテキストの言語が宣言されている必要があり、日本語のデッキなら要素かその祖先に`lang="ja"`が要る。Peithoの出力は`<html lang="en">`で固定なので、テーマのCSSで`word-break: auto-phrase`を指定しても、そのままではどのスライドにも効かない。

逃げ道が全くないわけではない。`peitho build`の出力なら生成後のHTMLを書き換えて`lang="ja"`にできるし、レイアウトHTMLは属性がそのまま出力されるので、カスタムレイアウトのルートの`<section>`に`lang="ja"`を書けばスライド単位で言語を宣言することもできる。ただ、前者は`peitho preview`や`peitho present`には効かないし、後者は言語を伝えるためだけにレイアウトを自前で持つことになる。デッキの言語はレイアウトの都合ではなくデッキ自体の属性なので、frontmatterで宣言できるのが筋だろう、ということで今回の機能を入れた。

---

## 使い方

デッキのfrontmatterに1行足すだけ。

```yaml
---
lang: ja
---
```

値はBCP 47スタイルの言語タグ（`en`、`ja`、`zh-Hans`など）。省略時のデフォルトは`en`で、既存のデッキの出力はバイト単位で変わらない。

`lang`が付くのは、スライドのHTMLを本体として描画するページ全部。配布用の`index.html`、PDFエクスポート、`peitho lint`の検査用HTML、present、preview、発表者ツールの6種類。一方、スマホリモコンのページとpreviewのエラーページはPeitho自体のUIで中身が英語なので、`lang="en"`のままにしている。

---

参考:

- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [前回の記事: Peithoにスライドのはみ出し検査（peitho lint）を入れた](/blog/2026/07/18/2/)
