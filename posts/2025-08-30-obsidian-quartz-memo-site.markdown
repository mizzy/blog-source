---
title: ObsidianとQuartz 4でつくる雑なメモサイト
date: 2025-08-30 12:30:00 +0900
---

今までGistやはてなブログに書いていたホテルのワークチェアについてのメモだけど、もっと雑に更新したいし、行ったことあるホテルでは実際に椅子がどうだったのか、といったあたりもまとめておきたいな、と思った。

<!-- markdownlint-disable MD013 -->
<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">これ、もっと雑に更新したいし、行ったことあるホテルでは実際に椅子がどうだったのか、といったあたりもまとめておきたいな、と思ったので、ブログからは削除して https://t.co/KoTUjAm69A に内容を移した https://t.co/41tPIx2dMx</p>&mdash; mizzy (@gosukenator) <a href="https://twitter.com/gosukenator/status/1961037784397021260?ref_src=twsrc%5Etfw">August 28, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<!-- markdownlint-enable MD013 -->

なので、別サイトで管理することにした。

どうやって作ろうかな、と考えたところで、ObsidianとQuartz 4の組み合わせを使うことにした。

<!-- markdownlint-disable MD013 -->
<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">ObisidianでMarkdownを書いてQuartz 4でGitHub PagesにPublishするようにした https://t.co/RwjecYnrw5</p>&mdash; mizzy (@gosukenator) <a href="https://twitter.com/gosukenator/status/1961040479656849472?ref_src=twsrc%5Etfw">August 28, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<!-- markdownlint-enable MD013 -->

---

## ObsidianとQuartz 4の組み合わせ

[Obsidian](https://obsidian.md/)は、Markdownベースのナレッジベースアプリ。ローカルで動作するので、データは自分の手元に置いておける。

[Quartz 4](https://quartz.jzhao.xyz/)は、Obsidianで書いたMarkdownファイルを静的サイトに変換してくれるツール。GitHub Pagesにデプロイできる。

この組み合わせの何がいいかというと、

- Obsidianで書いたメモをそのまま公開できる
- GitHub Actionsで自動デプロイできる

といったあたり。

---

## セットアップ

Quartz 4のセットアップは、ghコマンドとghqコマンドを使うと簡単。

```shell
gh repo fork jackyzha0/quartz --clone=false
ghq get git@github.com:YOUR_USERNAME/quartz.git
cd $(ghq root)/github.com/YOUR_USERNAME/quartz
npm i
npx quartz create
```

あとは、`content`ディレクトリにObsidianのVaultを配置するか、シンボリックリンクを張るかすれば良い。

自分の場合は、Obsidianで管理してるVaultの`publish`ディレクトリだけを公開したかったので、Makefileを使って`content`ディレクトリにコピーするようにした。

実際のコードは[hotel-memo/hotel-memo.github.io](https://github.com/hotel-memo/hotel-memo.github.io)にある。

---

## 雑なメモサイトとしての使い心地

実際に[ホテルメモサイト](https://hotel-memo.github.io/)を作って運用してみたところ、かなり使い心地が良い。

Obsidianで書いて、`make serve`でローカルでプレビュー確認、`make publish`で公開、という感じで手軽に運用できる。ブログほど構えずに、本当に雑なメモとして書ける。

---

参考：

- [Obsidian](https://obsidian.md/)
- [Quartz 4](https://quartz.jzhao.xyz/)
- [ホテルメモ](https://hotel-memo.github.io/)
