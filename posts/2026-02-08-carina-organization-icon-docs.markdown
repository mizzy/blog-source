---
title: Carina organization移行とアイコン・ドキュメント整備
date: 2026-02-08 14:12:38 +0900
---

[Carina](https://github.com/carina-rs/carina)のリポジトリを個人アカウント（mizzy）から[carina-rs](https://github.com/carina-rs) organizationに移した。あわせて、アイコンの作成と[ドキュメントサイト](https://carina-rs.github.io/)の公開もやった。

---

## organizationへの移行

これまで[mizzy/carina](https://github.com/mizzy/carina)として個人アカウント配下に置いていたが、[carina-rs/carina](https://github.com/carina-rs/carina)に移した。

移行の主な理由は、ドキュメントサイトをGitHub Pagesでホストしたかったから。GitHub Pagesでorganizationサイト（`carina-rs.github.io`）を使えば、プロジェクト専用のドメインでドキュメントを公開できる。organization名は`carina-rs`にした。Rustプロジェクトでよくある命名だし、`carina`という名前はすでに取られていたので。

GitHubのリポジトリ移行機能を使えば、旧URLからのリダイレクトも自動で設定されるので、既存のリンクが壊れる心配もない。

---

## アイコンの作成

ChatGPTにお願いしてアイコンをつくってもらった。りゅうこつ座（Carina）の船の竜骨と、カノープスをイメージしたデザイン。カノープスはりゅうこつ座のα星で、シリウスに次いで全天で2番目に明るい恒星（太陽を除く）。ただ、日本では南の空の低い位置にしか見えないのでマイナーな印象がある。organizationのアバターと、リポジトリのREADMEのヘッダーにも使っている。

---

## ドキュメントサイト

[mdBook](https://rust-lang.github.io/mdBook/)を使ってドキュメントサイトをつくり、GitHub Pagesでホストするようにした。プロバイダのリソースドキュメントはCloudFormationスキーマから自動生成し、それ以外はClaude Codeにつくってもらった。

[Introduction - Carina Documentation](https://carina-rs.github.io/)

現時点では、Carinaの概要や主要機能の説明、クイックスタート、プロバイダのドキュメントなどを載せている。開発が進むにつれて、ドキュメントも充実させていく予定。

mdBookはRust製のドキュメントジェネレーターで、Markdownから静的サイトを生成してくれる。Rust製ツールのドキュメントによく使われているらしい。

---

## 関連リンク

- [carina-rs/carina: A strongly typed infrastructure management tool](https://github.com/carina-rs/carina)
- [Introduction - Carina Documentation](https://carina-rs.github.io/)
