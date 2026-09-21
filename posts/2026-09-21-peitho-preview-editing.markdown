---
title: Peithoのpreviewでスピーカーノートとスライド本文を直接編集できるようにした
date: 2026-09-21 11:03:49 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の`peitho preview`に、ブラウザ上で直接テキストを直す機能を入れた。1.30.0でスピーカーノート（[PR #496](https://github.com/mizzy/peitho/pull/496)ほか）、1.32.0でスライド本文（[PR #555](https://github.com/mizzy/peitho/pull/555)ほか）。どちらも書き戻し先はMarkdownのソースそのもので、`dist/`には一切入らない。

35秒のデモ。オーバービュー、単一スライド表示、スピーカーノートの入力、そしてスライドの箇条書きと見出しをその場で直すところまで。

<video src="/images/2026/09/peitho-preview-editing.mp4" controls muted playsinline style="max-width:100%"></video>

できるのは既にある要素の修正だけで、要素の追加や削除はできない。プレビューで全体感を見ながら微調整するのが目的なので、これで十分だと考えている。
