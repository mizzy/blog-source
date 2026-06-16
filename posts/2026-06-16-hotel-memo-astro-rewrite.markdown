---
title: ホテルメモサイトをAstroで作り直した
date: 2026-06-16 12:24:46 +0900
---

去年の8月に[ObsidianとQuartz 4でつくる雑なメモサイト](/blog/2025/08/30/1/)というエントリを書いて、[ホテルメモ](https://hotel-memo.github.io/)というサイトをObsidian + Quartz 4で運用していたのだけど、これをAstroで作り直した。リポジトリは元と同じ[hotel-memo/hotel-memo.github.io](https://github.com/hotel-memo/hotel-memo.github.io)で、サイトのURLも[https://hotel-memo.github.io/](https://hotel-memo.github.io/)のまま。

ビフォーアフターはこんな感じ。

*Before（Quartz 4）*
![Quartz 4で作っていた頃のホテルメモ](/images/2026/06/hotel-memo-quartz.png)

*After（Astro）*
![Astroで作り直したホテルメモ](/images/2026/06/hotel-memo-astro.png)

---

## 作り直した動機

ひとつめは、見た目の話。元々Obsidianでメモしていたものをそのまま公開したかったので、Obsidian Publishのクローン的な位置付けのQuartzを選んだ、という流れだった。ただ、別に見た目までObsidian Publishに寄せたかったわけではなくて、特に右サイドのグラフビューは自分のユースケース的に不要なのに、画面の幅をそれなりに占有していてうっとうしかった。コンフィグでいじれる範囲に限界があって、踏み込もうとするとQuartzのコードに手を入れることになりそうで、upstreamと乖離していくのが懸念だった。

もうひとつは、Obsidianを使う必然性が無くなったこと。最初にこの仕組みを作ったときはメモを自分でObsidianに書いていたので、Obsidian → Markdown → Quartzという流れに意味があった。でも今はメモの中身も生成AIに書いてもらうことが多くて、AIが書いたMarkdownを最終的にGitHubのリポジトリに置ければそれでいい、という状態になっている。Obsidianを経由する必要が無い。

それなら、ビルドはもっと素直な静的サイトジェネレータでやって、デザインやレイアウトも自分で決められる方が気持ちいい。ということでAstroで書き直すことにした。

---

## ビジュアルデザインはClaude Designにやってもらった

雑なメモサイトとはいえ、見た目がそれっぽくないとモチベーションも上がらないので、ビジュアルデザインは[Claude Design](https://www.anthropic.com/news/claude-design-anthropic-labs)にやってもらった。

まず最初にClaude CodeにAstroで雑に作ってもらい、Claude Designには「ホテルのワークチェアについてのメモサイトを作りたい」みたいな感じで方向性を伝えつつ、Claude Codeが生成したファイルを渡すと、いくつかデザインのパターンを出してくれる。そこから気に入ったものを選んで、何度かやり取りして方向性を固めていった。

そこからはページ種別ごとに繰り返し作業で、トップページ、地域別ページ、ホテルブランドページ、ホテルページ、部屋ページといった単位でClaude Designにコンテンツやデザインファイルを読み込ませつつデザインを提案してもらい、Claude Code向けの実装指示書を作ってもらって、それをClaude Codeに渡して実装する、という流れ。自分がやり方を知らないだけかもしれないけど、ローカルファイルを直接編集することはできないようなので、「ここを直したい」となるたびにClaude Designで作成した指示書をClaude Codeに渡し直す必要があって、そこは少し面倒だった。

---

参考：

- [ホテルメモ](https://hotel-memo.github.io/)
- [hotel-memo/hotel-memo.github.io](https://github.com/hotel-memo/hotel-memo.github.io)
- [Astro](https://astro.build/)
- [Claude Design](https://www.anthropic.com/news/claude-design-anthropic-labs)
