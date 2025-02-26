---
title: Paperboy's engineer evaluation system その後
date: 2012-08-20 23:59:43 +0900
---

[Paperboy's engineer evaluation system](/blog/2012/02/29/1/) で書いた、ペパボの技術者評価制度、2回目の評価が完了しました。

前回は、

 1. シニア、またはアドバンスドシニアに上がりたい人には、自ら立候補してもらう。
 1. 立候補する人は、定められたフォーマットにしたがって、自分がそのポジションにふさわしいと思う理由や実績について Markdown で書き、指定した Git リポジトリに push する。（「定められたフォーマット」と言っても、最初に名前、次に希望のポジションを書いてもらうだけで、それ以外は自由。）
 1. 文書提出後、一人一人と面談を行う。
 1. 文書の内容と面談の結果にもとづいて、各人が提出した文書の末尾に、結果（通過 or 不通過）、評価ポイント、今後期待すること、を評価者が追記し、git push する。

といった流れで、今回も基本的には同じ流れなのですが、2. の部分のやり方を変え、社内の Git リポジトリに push してもらう代わりに、GitHub 上にある文書提出用プライベートリポジトリを fork して、pull request を送ってもらう形にしました。

pull request をもらったら、一次評価者である自分が内容を確認し、ここをもう少し詳しく書いて、とか、こういうこともやってたから、盛り込んだらいいんじゃない、的なことを、pull request のコメントでやりとりして、修正してもらったらまた push、ってなことをやって、提出文書をブラッシュアップしてもらい、面談が終わったらマージ、fork 元のリポジトリにマージされた各人の提出文書に対して、評価結果とコメントを書き込む、という流れでした。

この、いったん提出してもらって、こちらから指摘して直してもらう、というのを、前回は個別に IRC とか口頭で話して、といった形でやってたんですが、今回は GitHub を利用することで、コミットの特定の行に対してピンポイントでコメントをつけることができて、すべて GitHub 上で完結できて便利でした。

また、GitHub のタイムラインを IRC にも流していたりするので、そのやりとりもすべて全社員が見る IRC チャンネルに流れる、という形で、よりオープンな評価ができたかな、と思っています。
