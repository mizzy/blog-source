---
title: Git scrapingでヒルトンの宿泊料金/ポイントをウォッチ
date: 2025-01-30 13:25:16 +0900
---

自分はHilton Honors会員かつ[HPCJ](https://www.hpcj.jp/)会員なので、ポイントを利用する場合やHPCJ対象外ホテルを予約する場合はオフィシャルサイトから、ポイントを利用せずHPCJ対象ホテルを予約する場合はHPCJサイトから予約している。

どちらのサイトで予約する場合でも、前日までキャンセル可能なFlexible Rateで予約をしている。（HPCJサイトでの予約は必然的にFlexible Rateになる。）

ホテルの宿泊料金は変動するので、予約した後で自分が予約した時の料金より安くなることがよくある。なので、予約後も料金を時々チェックして、予約時より安くなっていたらキャンセルして予約し直す、といったことをしている。

そこで、[PlaywrightでGit scraping](/blog/2025/01/28/1/)という記事で紹介した[git-scraping-playwright-template](https://github.com/mizzy/git-scraping-playwright-template)をベースにして、指定した日程のヒルトン宿泊にかかる料金やポイントを自動でウォッチできるやつをつくってみた。

<a href="https://github.com/mizzy/git-scraping-hilton"><img src="https://opengraph.githubassets.com/efbe89083ae3d67f091dde51fe157d2da35426dc8300f8baab15b6c14eeb82f5/mizzy/git-scraping-hilton" height="200" /></a>

config.tsで以下のようにウォッチしたいヒルトンホテルの場所や日程を指定する。

<script src="https://gist.github.com/mizzy/bb16cb6f2b0de94f25534c1b60c23232.js"></script>

今のところ、福岡、お台場、有明しか対応していない。対応ホテルを増やしたい場合は、config_type.tsの以下の部分に追加する。

<script src="https://gist.github.com/mizzy/88978b078626b9c464648f451f01b20a.js"></script>

各ホテルの料金やポイントは、`https://www.hilton.com/en/book/reservation/rooms/?ctyhocn=FUKHIHI&arrivalDate=...`
というURLから取得しているが、`ctyhocn=FUKHIHI`にホテルコードらしきものが入っているので、ロケーション名とホテルコードのマッピングを上のコードで行っている。

ポイントは「Pay with Points」に表示されているもの、料金は「Quick Book」のところに表示されているFlexible Rateの料金を取得している。

![](/images/2025/01/hilton-odaiba.png)

同じ日程、同じ部屋タイプでも様々な料金プランがあるが、基本的にはFlexible Rateと連動してるはずなので、Flexible Rateをウォッチしておけば、価格の変動はわかるはず。

より正確な情報を得ようとするならば、自分の場合、ポイント、Flexible RateにHonors Discountが適用された価格、HPCJ価格（Flexible Rate * 0.75）の3つがあれば良いので、次はこれらの料金を取得できるよう改善する予定。
