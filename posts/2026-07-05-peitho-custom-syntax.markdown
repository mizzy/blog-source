---
title: Peithoでカスタムシンタックスハイライトができるようにした
date: 2026-07-05 21:00:00 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の話をここ数日続けて書いているけど、今日もうひとつ機能を入れたのでその話。デッキと同じディレクトリに`syntaxes/`を置くと、そこに入れた`.sublime-syntax`ファイルをシンタックスハイライトに使ってくれる、という機能。

Peithoが使っている[syntect](https://github.com/trishume/syntect)のデフォルトのシンタックスセットには、当然マイナーな言語や独自DSLは入っていない。Peithoは言語タグの付いたコードブロックのハイライトに未知の言語が指定されるとビルドエラーにする（黙って何もハイライトしないよりは気づけたほうがいい、という方針）ので、この機能が無いとそういう言語のコードブロックはタグなしで書くしかなくて、当然色は付かない。

---

## syntaxes/を置くだけ

やることはシンプルで、デッキファイルと同じディレクトリに`syntaxes/`ディレクトリを置いて、そこに`*.sublime-syntax`ファイルを入れておくだけ。

```
deck.md
syntaxes/
  crn.sublime-syntax
layouts/
css/
```

Peithoはビルド時にこのディレクトリを自動で見にいって、中の`.sublime-syntax`ファイルを組み込みのシンタックスセットに追加する。あとはコードブロックに`crn`と書けば普通にハイライトされる（言語タグは`.sublime-syntax`の中で宣言する`name`や`file_extensions`で決まる）。

````markdown
```crn
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
}
```
````

CLIフラグは足していない。Peithoは`layouts/`や`css/`もデッキと同じディレクトリにあれば自動で拾う規約になっていて、それと同じ形にした。

実際にこの機能を使って[Carinaを紹介するデッキ](https://decks.gosu.ke/carina/)をビルドしてみたのがこちら。`.crn`のブロックにちゃんと色が付いている。

![Peithoでハイライトされた.crnコードブロック](/images/2026/07/peitho-custom-syntax-crn.png)

`.sublime-syntax`はYAMLで書ける形式で、既存の言語であればGitHubを漁ればだいたい誰かが書いたものが見つかる（syntectはそれをそのまま読める）。独自DSLだったら自分で書くことになるけど、普通のYAMLファイル1枚で済む。

CSSも基本的には追加不要。Peithoはsyntectのscope名（`keyword`、`string`、`comment`、`storage`等）をそのまま`hl-*`クラスに変換して吐いていて、デフォルトテーマがその基本scopeに色を当てている。`.sublime-syntax`側で標準的なscope名を使っていれば、そのまま色が付く。

---

## 壊れたファイルは黙って落とさない

`.sublime-syntax`ファイルの読み込みでミスがあったとき（YAMLがパースできない、必須フィールドが無い等）は、行番号つきのビルドエラーにするようにした。

`SyntaxSetBuilder::add_from_folder`は素直に使うと壊れたファイルを黙って読み飛ばすんだけど、それをやってしまうと「シンタックスを追加したはずなのに効いていない」という状況になって、原因の特定に時間がかかる。ここは「黙って何かが起きる」経路を残さない、というPeithoの他の部分と同じ方針にした。ビルドを止めれば、少なくとも書いた本人はその場で気づける。

---

## 参考

- [PR #122](https://github.com/mizzy/peitho/pull/122) — 今回のシンタックスハイライトまわりの変更
- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [syntect](https://github.com/trishume/syntect) — Rust製のシンタックスハイライタ。ArgusやIrisと同じものを使っている
- [Sublime Text: Syntax Definitions](https://www.sublimetext.com/docs/syntax.html) — `.sublime-syntax`の書き方
