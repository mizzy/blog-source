---
title: 社会保険料額情報HTMLジェネレーター
date: 2025-05-24 18:36:10 +0900
---

社会保険料額情報、電子データで受け取る申請を行って、無事受理された。

[電子データで受け取れる各種情報・通知書の詳細｜日本年金機構](https://www.nenkin.go.jp/denshibenri/online_jigyousho/denshidata/tsuchisho.html)

申請時に、事業所整理記号をどう入力すれば良いのかよくわからず、何度かリジェクトされたけど。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">解決した。前2桁は都道府県コードを入れる必要があった。申請フォームに様式記入要領PDFへにリンクがあり、それを見ると書いてあった。 <a href="https://t.co/bfWQzaUiqL">https://t.co/bfWQzaUiqL</a></p>&mdash; mizzy (@gosukenator) <a href="https://twitter.com/gosukenator/status/1926157537159610449?ref_src=twsrc%5Etfw">May 24, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

電子データはZIPファイルになっていて、中身を見るとXMLファイルとXSLファイルで構成されている。IEではそのまま閲覧できるようだが、Microsoft EdgeやSafariの場合は、ブラウザの設定変更が必要になる。

- [InternetExplorer以外のブラウザでの決定通知書の閲覧方法について｜日本年金機構](https://www.nenkin.go.jp/denshibenri/oshirase/zenpan/20220610.html)
- [機構から送付されるXML形式の通知書（公文書）の開き方を教えてください。｜日本年金機構](https://www.nenkin.go.jp/faq/denshi/e-gov/koubunsho_hirakikata.html)

また、普段使いしてるChromeでは、そのままさくっと表示する方法がなさそうで、以下のサイトを参考に、XMLとXSLからHTMLに変換して表示する方法を試してみた。

- [電子公文書の XML と XSL を Python で１つのHTMLに変換する方法 - ガンマソフト](https://gammasoft.jp/blog/convert-xml-and-xsl-to-one-html-by-python/)
- [電子公文書(XML+XSL)をChromeで表示する](https://zenn.dev/sora_kumo/articles/xsl-viewer-html)

この方法でばっちり表示されたのだが、PDFで保存した場合にもきれいにレイアウトしたい、と思い、[Claude](https://claude.ai/)に社会保険料額情報HTMLジェネレーターつくってもらった。

ジェネレーターはひとつのHTMLファイルで構成されていて、開くとこのような画面になる。

![](/images/2025/05/social-insurance-premium-amount-generator-0.png)

社会保険料額情報のXMLが含まれるZIPファイルをアップロードするとこのようになる。

![](/images/2025/05/social-insurance-premium-amount-generator-1.png)

プレビュー表示するとこのようになり、PDFにするとA4横一枚におさまるレイアウトになっている。

![](/images/2025/05/social-insurance-premium-amount-generator-2.png)

ソースコードはこちら → https://github.com/mizzy/social-insurance

こちらで実際に試せます → https://mizzy.org/social_insurance_premium_amount_generator.html

社会保険料額情報だけしか対応していないので、保険料納入告知額・領収済額通知書等にも対応予定。
