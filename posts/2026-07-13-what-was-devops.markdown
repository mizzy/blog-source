---
title: DevOpsとは何だったのか
date: 2026-07-13 23:08:56 +0900
---

[SRE NEXT 2026の記事](/blog/2026/07/13/1/)で、maruさん、ゆううきさん、Moriさんとの立ち話から、こんなポストをした、ということを書いた。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr"><a href="https://t.co/mGZbILmQRK">https://t.co/mGZbILmQRK</a> にある「開発の時系列上の役割と、技術レイヤーの話が混ざっている」という話、maruさん、yuukiさん、moriさんと立ち話してた時に出た話で、DevOpsもそうだよね、という話になったのだけど、DevOpsも元々は、組織とか部門の話のはずなんだよね</p>&mdash; mizzy (@gosukenator) <a href="https://x.com/gosukenator/status/2076516038792114264?ref_src=twsrc%5Etfw">July 13, 2026</a></blockquote> <script async src="https://platform.x.com/widgets.js" charset="utf-8"></script>

整理してブログ記事にしてもいいかもな、たぶん面倒なのでやらない、と書いたけど、songmuさんのこのポストに触発されたこともあって、書くことにした。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">小文字のdevopsという考え方、僕は素敵だと思っているのだけど、あまり語られているのを見たことはない</p>&mdash; songmu (@songmu) <a href="https://x.com/songmu/status/2076556588610162714?ref_src=twsrc%5Etfw">July 13, 2026</a></blockquote> <script async src="https://platform.x.com/widgets.js" charset="utf-8"></script>

小文字のdevopsとは何なのか、も含めて、DevOpsという言葉が元々何を指していたのか、それがどう変わっていったのか、という話。

---

## DevOpsの原点

DevOpsの起源としてよく挙げられるのは、次の3つの出来事。

まず2008年、トロントで開催されたAgile 2008で、Andrew Clay Shafer氏が「Agile Infrastructure」というBoFセッションを企画した。だけど反応があまりに少なくて、Shafer氏本人は自分のセッションに現れなかった。そして会場に足を運んだのはたった一人、Patrick Debois氏だった。Debois氏はそれで諦めず、廊下でShafer氏を捕まえて話し込んだ。これがきっかけで、二人はAgile Systems Administrationというグループを立ち上げた。

Agile Infrastructureというテーマが示す通り、二人の問題意識は、開発の世界で広まっていたアジャイルのやり方を、インフラ運用の世界にも持ち込めないか、というものだった。アジャイルのカンファレンスでこのテーマに人が集まらなかったこと自体が、当時の開発と運用のコミュニティの距離を物語っている。

次に2009年6月のVelocity 2009で、FlickrのJohn Allspaw氏とPaul Hammond氏が、有名な「[10+ Deploys per Day: Dev and Ops Cooperation at Flickr](https://www.youtube.com/watch?v=LdOe18KhtT4)」という発表をした。開発者と運用者が対立するのではなく、協力することで1日10回以上のデプロイを実現している、という話。

そしてこの発表に触発されたDebois氏が、2009年10月にベルギーのゲントで開催したのが、最初の[devopsdays](https://devopsdays.org/about/)。devとopsにdaysをつけてdevopsdays。DevOpsという言葉はこのカンファレンス名から生まれた。

この経緯からわかるのは、DevOpsは元々、アジャイルを運用の世界まで広げたい、開発部門と運用部門という組織の分断をどうにかしたい、という問題意識から生まれた言葉だ、ということ。技術スタックのどこかのレイヤーを指す言葉でも、開発の時系列上のどこかの工程を指す言葉でも、特定のツール群を指す言葉でもない。

ちなみに自分がこの言葉に出会ったのは2010年3月で、当時[DevOpsについての記事](/blog/2010/03/25/1/)を書いている。そこでは「開発者と運用者の間の壁を取り払うためのベストプラクティス」と自分なりにまとめていた。

---

## CAMS——初期の定義の輪郭

DevOpsとは何か、を初期に言語化したものとしては、John Willis氏が2010年7月に書いた「[What Devops Means to Me](https://www.chef.io/blog/what-devops-means-to-me)」がある。ここで示されたのがCAMS、すなわちCulture（文化）、Automation（自動化）、Measurement（計測）、Sharing（共有）の4つ。

注目したいのは、Cultureが先頭に来ていて、Automationは4つのうちの1つでしかない、というところ。ツールによる自動化はDevOpsの一部ではあるけど、それがDevOpsなのではない、ということが、この時点でははっきりしていた。

同じ2010年には、広まりはじめたDevOpsに対する批判も出てきていた。そのひとつ「[Incomplete Thought: The DevOps Disconnect](http://www.rationalsurvivability.com/blog/2010/05/incomplete-thought-the-devops-disconnect/)」という記事のコメント欄で、Velocity 2009の発表者本人であるJohn Allspaw氏がこう書いている。

> It's not abstraction. It's not even "infrastructure as code". It's not any single tool. It's not about provisioning. It's not about deployment. It's not about a job description or position. ... It *is* about the collaborative and communicative culture and the tools and process that arise from that culture. Nothing more.

抽象化のことではない。Infrastructure as Codeのことですらない。特定のツールのことでも、プロビジョニングやデプロイのことでも、職種や役職のことでもない。協力とコミュニケーションの文化、そしてその文化から生まれるツールやプロセスのことだ。それ以上ではない——と、「DevOpsではないもの」を徹底的に列挙した上で、文化の話であると断言している。

このコメント、自分は[2011年6月のDevOpsカンファレンスでの発表](/blog/2011/06/25/1/)で引用していて、さらに[2012年にはtumblrにもリポストしていた](https://mizzy.tumblr.com/post/37557718970/its-not-abstraction-its-not-even)。

---

## モノの名前になっていくDevOps

その後どうなったかは、みなさんご存知の通り。組織文化の運動だったはずのDevOpsという言葉は、具体的なモノの名前として消費されるようになっていく。というより、Allspaw氏が2010年の時点でわざわざ「ツールのことではない、職種や役職のことではない」と列挙して否定しなければならなかったこと自体が、当時から既にそう受け取る人がいた、ということの現れでもある。その流れは押しとどめられることなく、そのまま定着していった。

どうモノの名前になっていったのかは、2016年に出た『[Effective DevOps](https://www.oreilly.co.jp/books/9784873118352/)』（Jennifer Davis、Ryn Daniels著、日本語版は2018年）の5章「devopsに対する誤解とアンチパターン」が列挙する「よくある誤解」を見るとよくわかる。「devopsはチームである」「devopsは肩書だ」「devopsには認定資格が必要だ」「devopsはツールの問題だ」。部署名、職種名、資格名、ツール名。さらに「devopsとは自動化のことだ」という誤解も挙げられていて、CAMSでは4分の1でしかなかったAutomationがdevopsそのものと同一視されるところまで来ていたことがわかる。また「はじめに」にも「全部入りのdevopsだとかdevops-as-a-serviceといったものを提供することはない」という一節があって、売り買いされる商品としてのDevOpsも、当時すでに存在していたことがうかがえる。2016年の時点で、この流れはすでに一章を割いて反論しなければならないほど進行していた。そして今では、これらは誤解と指摘されることすらなく、当たり前のものとして定着している。

---

## 小文字のdevops

『Effective DevOps』本文中では一貫して、DevOpsではなく小文字のdevopsと表記されている。上に書いた5章のタイトルが「devopsに対する誤解とアンチパターン」なのもそのため。

これは意識的な選択で、「はじめに」には「Devops、devops、DevOps: どれがよいか」というコラムまであって、そこで理由が説明されている。要約すると、

- 表記についてはオンライン投票までやっていて、結果はDevOpsの圧勝だった。だけどDevOpsという表記は、企業の中でDevとOpsに重点が置かれていることの現れでもある。DevOpsではDevとOps以外が排除されてしまうので、DevSecOpsやDevQAOpsといった言葉を作る動きもある
- Twitterの#devopsハッシュタグは、こちら対あちらという対話のあり方を変えて、人を重視した持続可能な仕事の方法を実現したい人たちをつなぐために使われていた。それに倣った
- プロジェクトの成功には組織全体の人たちのインプットや知見、コラボレーションが必要で、組織固有の問題は開発チームと運用チームの対立に限らないかもしれない

という3点。その上で、コラムはこう締めくくられている。

> devopsは、排除ではなく開放のための運動だという私たちの考え方を反映させるために、本書全体を通じて意識的にdevopsという小文字表記を使うことにした。

つまり小文字のdevopsという綴りには、「これは特定の職種やチームやツールの名前ではなく、組織全体の文化的な運動である」という、原点の定義そのものが埋め込まれている。実際、同じ「はじめに」の中で、devopsという言葉自体もこう定義されている。

> devopsは情報のサイロを壊し、関係を観察し、チーム間で発生する誤解を解消するための反復的な取り組みを強調するプロフェッショナルで文化的な運動だ。

職種でもチームでもツールでもなく、あくまで「プロフェッショナルで文化的な運動」。

---

## 訳の問題でも、言語の問題でもない

ただ、結果を見れば、小文字のdevopsが定着することはなかった。もっとも、本のコラムにあった通りオンライン投票ではDevOpsの圧勝だったわけで、DとOを大文字にしたDevOpsという表記は当時から標準だった。そしてそれはその後も変わっていない。AWSのドキュメントも、Google CloudのDORAが長年出してきたState of DevOps Reportも、みんなDevOps表記だ。小文字表記を貫いたこの本の方が、最初からずっと例外だった。

冒頭の立ち話の流れで、こんなポストもしていた。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">DevOpsが「開発と運用」って訳されてるのが悪い気もするので、この訳語で日本に持ち込んだやつが悪い、って思ったけど、それは多分俺なんだよね。なので俺が悪いのか。でもそれ以外に訳しようもない気がするので、誰が訳してもこうなる気はする <a href="https://t.co/avBDi1JOsK">https://t.co/avBDi1JOsK</a></p>&mdash; mizzy (@gosukenator) <a href="https://x.com/gosukenator/status/2076520504828281047?ref_src=twsrc%5Etfw">July 13, 2026</a></blockquote> <script async src="https://platform.x.com/widgets.js" charset="utf-8"></script>

だけどこれは的を外している。まず、2010年3月の自分の記事を読み返すと、実際には「開発と運用」ではなく「開発者と運用者」という言葉で紹介していた。そしてそもそも、訳は関係ない。ここまで見てきた通り、訳が介在しない英語圏でも、まったく同じ誤解が広がっているのだから。

日本語だと、大文字と小文字の使い分けという慣習自体がないので、日本語版『Effective DevOps』が章タイトルまで小文字のdevopsを保っていても、その意図は本文の説明を読んで初めてわかる。だけど、綴りで意図を表現できる英語圏でも、その意図は広まっていないように見える。訳の問題でも、言語の問題でもない。

文化の運動という掴みどころのないものより、職種やツールや資格という、名乗れるもの・買えるものの方が流通しやすい。それだけの話なのだと思う。

---

## 別の言葉へと分解されていくDevOps

その後の流れを見ていくと、DevOpsという言葉が包含していたものが、それぞれ別の言葉として具体化されていったことがわかる。Velocity 2009のFlickrの発表で「ツール」として挙げられていた要素を、いま流通している言葉と突き合わせてみると、

- インフラの自動化 → Infrastructure as Code
- ワンステップビルド＆デプロイ → CI/CD
- 機能フラグ → フィーチャーフラグ（専用のSaaSまである）
- メトリックの共有 → オブザーバビリティ、DORAの4キーメトリクス
- IRCやIMのロボット → ChatOps

という感じで、ほぼすべてが独立した言葉になり、実践とツールと市場を持つ領域に育った。（発表のリストにはもうひとつ「共有されたバージョンコントロール」があるけど、これは当たり前になりすぎて、対応する言葉を挙げるまでもない。）Infrastructure as Codeのように、言葉自体は当時からあったものもある（先に引用したAllspaw氏の2010年のコメントにも出てくる）ので、言葉の新しさがポイントではない。DevOpsの文脈の中の一要素だったものが、それぞれ、DevOpsという言葉を持ち出さなくても語れる独立した領域になった、ということがポイント。発表の核心にあった「安定性をとるか、変化をとるか」というDevとOpsの対立も、SREのエラーバジェットという形で仕組みに落とし込まれた。この構図を端的に表しているのが、「class SRE implements DevOps」——SREはDevOpsという抽象クラスの実装である——という整理。GoogleのLiz Fong-Jones氏とSeth Vargo氏が2018年の動画シリーズで打ち出したフレーズで、同年の「[The Site Reliability Workbook](https://sre.google/workbook/how-sre-relates/)」の章タイトルにもなっている。DevOpsは、具体的な実装を持つ言葉たちの上に浮かぶ抽象概念、という位置づけになった。

そして、抽象概念だけになった言葉は、だんだん使われなくなっていく。Platform Engineeringの文脈では「DevOpsは死んだ」といった言説も見かけるようになった。2022年には「DevOps is dead, long live Platform Engineering!」という[Sid Palas氏のポスト](https://x.com/sidpalas/status/1580978289534914561)が大きな論争になったし、The New Stackの「[DevOps Is Dead. Embrace Platform Engineering](https://thenewstack.io/devops-is-dead-embrace-platform-engineering/)」という記事や、[同名のCNCFのウェビナー](https://www.cncf.io/online-programs/cncf-on-demand-webinar-devops-is-dead-embrace-platform-engineering/)もあった。象徴的なのはDORAの動きで、DORAはDevOps Research and Assessmentの頭字語で、毎年「Accelerate State of DevOps Report」という調査レポートを出していたのだけど、[2025年にレポート名を「State of AI-assisted Software Development」に変更した](https://dora.dev/insights/dora-2025-year-in-review/)。あわせてDORA自体も、頭字語ではない単独の名前ということになった。DevOpsという言葉は、レポートの名前からも、組織の名前からも消えたことになる。

---

## 問題は解決したのか

ただ、何が具体化されて、何が残ったのかを見てみると、別の言葉になったのはどれも、ツールやプラクティスや職種として名前を付けられる部分——名乗れる部分、売り買いできる部分——だということに気づく。CAMSで言えば、AutomationとMeasurementには立派な後継の言葉ができた。

じゃあ先頭にあったCulture、つまり部門間の分断や対立をどうにかする、という部分はどうか。候補が無いわけではない。チーム間の分断や認知負荷を正面から扱う[Team Topologies](https://teamtopologies.com/)や、blamelessなポストモーテムの系譜を引き継ぐLearning from Incidents、Resilience Engineeringは、Cultureの一部を引き受けていると言えると思う（ちなみにAllspaw氏自身が、Etsyを離れた後、[Adaptive Capacity Labs](https://www.adaptivecapacitylabs.com/)を共同創業してこの領域に進んでいる）。ただ、これらもやはり、本やカンファレンスやコンサルティングという、名乗れる・売り買いできる形になって流通している。Cultureですら、名乗れる形に整形された部分だけが生き延びている、と言えるのかもしれない。

じゃあ、これらの言葉によって、部門間の分断という問題自体は解決したのかというと、そうは見えない。

たとえばDevSecOps。この言葉が生まれたのは2013年頃とされていて、DevOpsの誕生からわずか4年ほどで、セキュリティを仲間に入れるための新しい言葉が必要とされていたことになる。部門の壁を壊す運動が機能しているのなら、セキュリティも当然そこに含まれているはずだ。先に見た通り、『Effective DevOps』の著者たちが小文字のdevopsを選んだ理由のひとつは、まさにこの語を作る動きへの応答だった。そしてDevSecOpsという言葉が[今も使われ続けている](https://www.practical-devsecops.com/devsecops-statistics-2026/)のは、壁が残り続けていることの現れだと思う。

Platform Engineeringもそう。DevOpsの実践は「You build it, you run it」のような、開発者がインフラ運用まで担う方向に進んだ。その結果、今度は開発者側にインフラ運用の認知負荷が寄りすぎて、それをプラットフォームチームが引き受け直そう、というのがPlatform Engineering。この「プロダクトチームに責任が移った結果、インフラへの認知負荷が増えたので、プラットフォームで引き受け直す」という認識は、[CNCFのPlatforms White Paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/)にも書かれている。インフラ運用を誰がどこまで担うのか、という分担の問題が、名前と形を変えて続いている、とも言える。そして、冒頭のポストで触れたmaruさんの[SRE NEXT 2026の参加レポート](https://sizu.me/maruloop/posts/kiiwa97nez3u)には、開発の時系列上の役割と、技術レイヤーと、組織上のロールは本来別々の軸である、という話が書かれていて、軸を混ぜてしまうと何が起きるかについて、こんな一節がある。

> この二つを混ぜてしまうと、SREは「運用の問題をソフトウェアエンジニアリングで解決するプラクティス」ではなく、「インフラレイヤーを担当する職種」になってしまいます。

これはまさに、DevOpsに起きたことと同じことが、SREという言葉でも起きている、ということだと思う。文化やプラクティスの話が、職種や技術レイヤーの名前として消費されていく。そしてそれが混乱として認識されるくらいには、組織の話と技術の話を切り分けて語るための語彙が、いまも足りていない。

[2011年のDevOpsカンファレンスの記事](/blog/2011/06/25/1/)を読み返すと、帰り際に[marqsさん](https://x.com/marqs)と、みんな同じような問題意識を抱えていたのに、それを適切に表す言葉がいままでなかった、DevOpsという言葉が与えられたことで、それが共通認識としてはっきり浮かび上がった、それって素晴らしいことだよね、という話をして握手した、と書いてあった。DevOpsという言葉の一番の価値は、名前のなかった問題に名前を与えたことだったのだと思う。そして、その名前がモノの名前として消費し尽くされた今、問題は解決されないまま、また名前を失いつつあるのかもしれない。

---

参考:

- [The Incredible True Story of How DevOps Got Its Name - New Relic](https://newrelic.com/blog/news/devops-name)
- [The Origins of DevOps: What's in a Name? - DevOps.com](https://devops.com/the-origins-of-devops-whats-in-a-name/)
- [10+ Deploys per Day: Dev and Ops Cooperation at Flickr (YouTube)](https://www.youtube.com/watch?v=LdOe18KhtT4)
- [10+ Deploys per Day 発表スライド (PDF)](https://paulhammond.org/2009/06/velocity/10deploys.pdf)
- [What Devops Means to Me - Chef Blog](https://www.chef.io/blog/what-devops-means-to-me)
- [about devopsdays](https://devopsdays.org/about/)
- [Effective DevOps ―4本柱による持続可能な組織文化の育て方 - O'Reilly Japan](https://www.oreilly.co.jp/books/9784873118352/)
- [DevOps (2010年3月に自分が書いた記事)](/blog/2010/03/25/1/)
