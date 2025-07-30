---
title: Twilog MCP Alternativeの実装
date: 2025-07-31 05:26:59 +0900
---

以下のポストに対して、Twilog MCPを利用するために課金していた一ユーザーとして思うところがあり、課金をやめて代わりになる仕組みを自前実装することにした。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">まったく書く予定じゃなかったnoteを書きました！やってしまいました＆やってしまったのは仕方ない＆やるしかない<br><br>7月30日16時過ぎから発生しているTwilogのデータベース障害について｜Togetter（トゥギャッター ） <a href="https://t.co/1CqVnqUu5x">https://t.co/1CqVnqUu5x</a></p>&mdash; yositosi (@yositosi) <a href="https://twitter.com/yositosi/status/1950577394219434263?ref_src=twsrc%5Etfw">July 30, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ググってみると、アーカイブデータをDuckDBにインポートして、MCPで閲覧する、という仕組みを構築されている方がいらっしゃった。

[DuckDB + Claude Desktop + MCP で X（Twitter）のアーカイブデータを閲覧する](https://zenn.dev/tenkao/scraps/96996339ddf4c1)

これを参考に、TwitterのアーカイブデータをDuckDBにインポートするプログラムをClaude Codeに実装してもらった。

[mizzy/tweetduck: Twitter Archive to DuckDB Importer - Extract and import Twitter archive data (2025 format) into DuckDB for analysis](https://github.com/mizzy/tweetduck)

これと[ktanaka101/mcp-server-duckdb: A Model Context Protocol (MCP) server implementation for DuckDB, providing database interaction capabilities](https://github.com/ktanaka101/mcp-server-duckdb)を組み合わせたら、完全にTwilog MCP Alternativeになった。

<img src="/images/2025/07/duckdb-mcp.png" alt="" width="50%" height="50%"/>

アーカイブデータの更新は手動になってしまうが、自分の用途的には、直近のツイートが入ってなくても問題ないので、これで十分。
