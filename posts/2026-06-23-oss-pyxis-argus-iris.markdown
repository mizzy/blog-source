---
title: 最近つくっているOSS 3つ - Pyxis / Argus / Iris
date: 2026-06-23 21:03:56 +0900
---

最近つくっているOSSをまとめて紹介しておく。

---

## Pyxis

[Pyxis](https://github.com/mizzy/pyxis)はLLM推論エンジン。[candle](https://github.com/huggingface/candle)も[llama.cpp](https://github.com/ggml-org/llama.cpp)も使わずに、トランスフォーマーのパイプライン全体（アテンション、KVキャッシュ、RoPE、サンプリング）を自前実装する、というプロジェクト。

つくりはじめたきっかけは、[コーディングのためのローカルLLM勉強会 in 福岡](https://connpass.com/event/395614/)に参加して、[@monochromegane](https://x.com/monochromegane)さんと[@kis](https://x.com/kis)さんと懇親会で話したこと。Carinaのplan表示をわかりやすくしたいという話を@monochromeganeさんとしていたら、生成AIに解説させるというアプローチを提案してもらった。その場ではGitHub ActionsでAIを呼び出す方向で考えていたけど、Carina自体に機能を持たせればいいんじゃないかと思った。また、@kisさんと@monochromeganeさんが推論エンジンについて会話しているのを横で聞いていて「ぜんぜんわからない」と言ったら、@kisさんから「6回実装してみればわかりますよ」と言われた。じゃあ、Carina自体にplan等をAIに解説させる機能を組み込んで、その推論エンジンも自前で実装してしまえ、と思ってつくりはじめた。

今の状態は、safetensorsのパーサーと、テンソル一覧のテーブル表示あたりが動いている段階。safetensorsファイルからテンソルデータをF32/BF16/F16として読み込む部分も実装中。推論エンジンの仕組みを学習する、という目的もあり、Claude Codeに実装と解説をさせ、ひとつひとつ理解しながらなので、亀の歩みではある。

名前は羅針盤座（Pyxis）から。[Carina](https://github.com/carina-rs/carina)がりゅうこつ座で、どちらも、現在は4つの星座に分割されたアルゴ座（Argo Navis）の一部にあたる星座。CarinaのplanをAIが解説する、つまり羅針盤が船に方向を示す、という対比でつけた。

---

## Argus

[Argus](https://github.com/mizzy/argus)はシンタックスハイライトとgit diff表示を備えたページャー。

![Argusのスクリーンショット](/images/2026/06/argus-screenshot.png)

ファイルを指定して開くCLIツールで、以下のように使う。

```bash
argus <file>
argus --rev HEAD~1 <file>
argus --rev HEAD~3..HEAD <file>
```

未コミットの変更があるファイルを開くと、diff領域が自動的にハイライトされる。`--rev`を指定すれば、コミット済みの変更をdiffとして表示することもできる。変更のかたまり（チェンジグループ）間を`n`/`N`でジャンプできたり、インクリメンタルサーチが使えたりする。また、word-level diffも実装している。

git diffだと差分周辺しか表示されないけど、それより広い範囲でコードを確認したいことがよくある。そのためにエディタを開いたりlessを使ったりしていたけど、最近コードを書くのはターミナルだけで済んでいるので、そのためにエディタを開くのもだるい。かといってlessはファイルをそのまま表示するだけで、差分が追いにくいし、シンタックスハイライトがないので読みにくい。ターミナルで差分表示するツールは色々あるけれど、どれも自分にはリッチすぎる。シンタックスハイライトと差分表示に対応したページャー程度のものがあればいいな、と思ってつくった。

名前はギリシャ神話の百の目を持つ巨人アルゴス（Argus）から。コードを隈なく見通すツールとしてしっくりくるし、アルゴス自身がアルゴ船を造った船大工でもあるので、[Carina](https://github.com/carina-rs/carina)（アルゴ船の竜骨座）との系譜もきれいにつながる。

---

## Iris

[Iris](https://github.com/mizzy/iris)はgitのdiff出力にシンタックスハイライトをかけるツール。

![Irisのスクリーンショット](/images/2026/06/iris-screenshot.png)
`git config --global core.pager iris`と設定すると、すべてのgitサブコマンドのページャーとして使えるようになる。

stdinからunified diffを読んで、変更されたファイルの拡張子から言語を検出し、シンタックスハイライトをかけた状態で出力する。diffのフォーマット自体（ヘッダー、`+`/`-`マーカー、ハンク情報）はそのまま残す。

シンタックスハイライトには[syntect](https://github.com/trishume/syntect)を使っている。カラーテーマは`IRIS_THEME`環境変数で指定できて、デフォルトは`base16-ocean.dark`。

```bash
export IRIS_THEME="Solarized (dark)"
iris --list-themes  # 利用可能なテーマ一覧
```

Argusを使っていて、同じようにgit diff等でもシンタックスハイライトされるといいな、と思ってつくった。こちらも既成ツールはあるけど、どれも見た目がリッチすぎてごちゃついて見えるので、gitの出力そのままにシンタックスハイライトするだけのシンプルなものが欲しくてつくった。

名前はギリシャ神話の虹の女神アイリス（Iris）から。虹の多色性がシンタックスハイライトとかかっているし、irisには眼の虹彩という意味もあるので、色と「見る」の両方に二重にかかっている。百の目を持つArgusと並べると、どちらも「見る／目」のモチーフが揃うのも気に入っている。
