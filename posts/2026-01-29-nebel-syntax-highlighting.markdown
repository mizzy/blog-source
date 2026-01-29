---
title: Nebelにシンタックスハイライト機能を追加した
date: 2026-01-29 18:55:03 +0900
---

最近このブログは、[Carina](https://github.com/mizzy/carina)の開発進捗報告の場と化して更新頻度が上がっているので、ブログ自体にも手を入れていっている。

2日前には[OGP画像自動生成機能を追加した](https://mizzy.org/blog/2026/01/27/2/)が、今回はコードのシンタックスハイライト機能を追加した。

---

## 実装内容

Markdownパーサとして使っている[goldmark](https://github.com/yuin/goldmark)に、[goldmark-highlighting](https://github.com/yuin/goldmark-highlighting)拡張を追加した。バックエンドには[Chroma](https://github.com/alecthomas/chroma)というGo製のシンタックスハイライターを使っている。

テーマは[Catppuccin Mocha](https://catppuccin.com/)を採用した。パステル調のダークテーマで、紫・青・緑・オレンジ・ピンクの柔らかい配色が特徴。ブログの背景が白なので、ダークテーマの方がコードブロックが目立つかな、と思って選んだ。

---

## 使い方

Markdownで言語を指定してコードブロックを書くと、自動的にハイライトされる。

たとえばJSONを書くとこうなる。

```json
{
  "name": "example",
  "version": "1.0.0"
}
```

Bashだとこう。

```bash
$ echo "Hello, World!"
```

---

## Carina言語のサポート

せっかくなので、開発中の[Carina](https://github.com/mizzy/carina)用のカスタムレクサーも追加した。Chromaはカスタムレクサーを定義できるので、独自言語のハイライトも可能。

```crn
backend s3 {
  bucket      = "my-carina-state"
  key         = "prod/carina.crnstate"
  encrypt     = true
  auto_create = true
}

let main_vpc = aws.vpc {
  name       = "main-vpc"
  cidr_block = "10.0.0.0/16"
}
```

キーワード、文字列、真偽値、リソースタイプなどが適切にハイライトされる。

---

## CSSの統一

言語指定ありのコードブロックはChromaでハイライトされるが、言語指定なしのコードブロックは素の`<pre>`タグで出力される。見た目を統一するため、`base.css`の`pre`スタイルもCatppuccin Mochaテーマに合わせた。

また、コードブロックのフォントサイズを本文と同じ16pxにして、読みやすさを向上させた。

```
言語指定なしのコードブロックも
同じダークブルー背景で表示される
```

---

## まとめ

- goldmark-highlighting（Chroma）でシンタックスハイライトを実装
- Catppuccin Mochaテーマでパステル調のダークなコードブロックに
- Carina言語用のカスタムレクサーを追加
- フォントサイズを本文と統一して読みやすく
- 言語指定なしのコードブロックもスタイルを統一
