---
title: Peithoのリハーサル機能で録音に対応した
date: 2026-09-19 09:06:17 +0900
---

自作プレゼンツール[Peitho](https://github.com/mizzy/peitho)の1.31.0で、リハーサル中のマイクの音声を、タイマーと同期した形で録音できるようにした（[PR #528](https://github.com/mizzy/peitho/pull/528)、[#531](https://github.com/mizzy/peitho/pull/531)）。

```sh
peitho present --rehearsal --audio
```

リハーサルが終わると、`peitho rehearsal`でこういう表が出る。

```
slide   key        entered   visits   total
#1      setup         0:00        1    0:48
#2      problem       0:48        1    1:07
#3      approach      1:55        1    2:40
total                                  4:35
audio   .peitho/rehearsals/rehearsal-20260719-135241.webm
```

3枚目は録音の1:55から始まっているので、そこを聞き直したければこう。

```sh
ffplay -ss 1:55 .peitho/rehearsals/rehearsal-20260719-135241.webm
```

---

## リハーサルモードのおさらい

リハーサルモード自体はブログに書いていなかったので、先に軽く説明しておく。

Peithoのデッキには、セクションと計画時間を書ける。

```markdown
<!-- {"key":"intro","section":"Setup","time":"3m"} -->
```

発表者ツールのアジェンダには、これを元にセクションごとの計画・実績・差分が出る（[タイマーの記事](/blog/2026/07/05/1/)で書いたやつ）。ただしこの実績は発表者ツールを閉じたら消える。`peitho present --rehearsal`で起動すると、これが`.peitho/rehearsals/`にJSONで残って、あとから`peitho rehearsal`で見返せる（[PR #318](https://github.com/mizzy/peitho/pull/318)）。何度か通してみて、時間配分を直していくためのもの。

---

## タイマーと同期した録音

録音するだけなら、QuickTimeなどで録っておいて、Peithoはタイムスタンプだけ出す、という手もある。ただそれだと、録音の開始とタイマーの開始を手で合わせることになる。タイマーを持っているのはPeithoなので、録音もPeithoがやれば、合わせる作業自体がなくなる。

`--audio`を付けると、発表者ツールのウィンドウでマイクを録音する。録音はタイマーに従属していて、タイマーが動き出したら録音開始、一時停止したら録音も一時停止、リセットしたら録音も破棄して録り直しになる。タイマーが進んでいる間だけ録音も進むので、タイマーが1:55のときに喋っていた内容は、録音ファイルの1:55のところに入っている。

冒頭の表の`entered`が、そのスライドに入った時刻。これはスライド単位のタイムラインを記録に足したことで出せるようになった（[PR #527](https://github.com/mizzy/peitho/pull/527)）。

再生用のUIや文字起こしは入れていない。JSONとWebMがあれば、whisperなりLLMなり外のツールに渡せるので、Peithoはそこまでやらなくていい、という判断。

---

## ブラウザで録ってサーバに送る

録音は、発表者ツールのブラウザの`MediaRecorder`で録っている。`MediaRecorder`は録り終えてからまとめて渡してくるのではなく、録音中に少しずつ音声データを渡してくるので、それをそのままサーバに送って、ファイルの末尾に足していく。タイマーを持っているのは発表者ツールなので、録音も同じページでやれば一時停止・再開にそのまま追従できる。送りながら保存しているので、途中でブラウザやサーバが落ちても、そこまでの録音は残る。

`pause()`/`resume()`で本当に隙間なくつながるのかは、実際のChromeで測った。10秒一時停止しても録音側に隙間はできず、録音の長さは「経過時間 − 録音開始位置」と30ms以内で一致した。

サーバ側で`ffmpeg`や`sox`を起動する案もあったけど、外部コマンドへの依存が増えるうえ、サーバ側のレコーダーは一時停止・再開に正確に追従できない。

---

## 応用例

リハーサルが終わると、`.peitho/rehearsals/`に同じ名前でWebMとJSONが並ぶ。

```
.peitho/rehearsals/rehearsal-20260719-135241.webm
.peitho/rehearsals/rehearsal-20260719-135241.json
```

WebMが録音そのもので、JSONのほうには冒頭の表の元になったデータ、つまりどのスライドに何ミリ秒の時点で入ったかが並んでいる。この2つが揃っているので、こんなことができそう。

- WebMをwhisperに渡して文字起こしする。タイムスタンプ付きで出せるので、JSONに入っているスライドごとの開始時刻と突き合わせれば、発言をスライド単位に切り分けられる
- 切り分けたテキストをLLMに渡して、時間を食っているスライドの喋りすぎている部分を削ってもらう
- 喋った内容をスピーカーノートに書き戻す
- 言い淀みや早口になっているスライドを特定する
- スライドに書いてあることと、実際に喋った内容がずれていないか突き合わせる

---

参考:

- [mizzy/peitho](https://github.com/mizzy/peitho)
- [デモサイト peitho.gosu.ke](https://peitho.gosu.ke/)
- [設計ドキュメント: Rehearsal audio and per-slide timeline](https://github.com/mizzy/peitho/blob/main/docs/specs/2026-09-18-rehearsal-audio-design.md)
- [Peitho発表者ツールのタイマー詳細](/blog/2026/07/05/1/)
