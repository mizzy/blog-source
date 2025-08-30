---
title: RigelというAI Coding Agentを開発している
date: 2025-08-30 21:31:14 +0900
updated: 2025-08-30 22:00:00 +0900
---

## きっかけ

tokuhiromさんの[ashronというAI Coding Agentを開発している](https://blog.64p.org/entry/2025/08/29/170302)というブログ記事や以下のポストを読んだ。

<!-- markdownlint-disable MD013 -->
<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">ashron っていう、Coding Agent を書いている - Blog::Amrta <a href="https://t.co/fKU4l0QTQq">https://t.co/fKU4l0QTQq</a></p>&mdash; tokuhirom (@tokuhirom) <a href="https://twitter.com/tokuhirom/status/1960847865967780266?ref_src=twsrc%5Etfw">August 29, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<!-- markdownlint-enable MD013 -->

Clineがいい、Cursorだ、いやClaude Codeだ、Codexだ、みたいな流行に振り回されたくないし、ローカルLLMをうまく扱えるコーディングエージェントもなかなか見つからない。CodexがローカルLLM対応してるけど、期待したとおりに動いてくれない。

なので、自分でAI Coding Agentを作るのがよいかもしれない、と考えていた。

そんな状況で、tokuhiromさんの記事を読んで背中を押された感じ。

というわけで、自分もRigelというAI Coding Agentを作り始めた。

---

## Rigelについて

[Rigel](https://github.com/mizzy/rigel)は、Goで実装しているAI Coding Agent。

主な特徴は以下の通り。

- ターミナルベースのチャットインターフェース
- 複数のLLMプロバイダーに対応（Anthropic Claude、Ollama）
- サンドボックスでの安全なコード実行

特にローカルLLMを扱えるようにOllamaに対応しているのがポイント。何も設定しない場合、デフォルトでローカルのOllama + gpt-oss:20bを利用するようになっている。エージェント実行中に、AnthropicかOllamaかを切り替えたり、モデルを切り替えたりもできる。

実際に動かすとこんな感じ。

![Rigelのスクリーンショット](/images/2025/08/rigel.jpg)

ちなみにRigelという名前はオリオン座の1等星から拝借しているけど、特に意味はない。リゲルは青白い星なので、色使いもそんな感じにしている。

---

## 今後の開発

とりあえずベースができて会話できる程度にはなったけど、まだコーディングエージェントとして使える状態にはなっていない。今後はツール周りと、コンテキストをどう扱うか、といったあたりが開発の中心になりそう。
