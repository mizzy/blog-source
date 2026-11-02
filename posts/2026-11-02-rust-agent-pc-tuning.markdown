---
title: 生成AIにRustで開発させるときのPC負荷対策
date: 2026-11-02 13:50:53 +0900
---

[Carina](https://github.com/carina-rs/carina)の開発は、ほぼClaude CodeとCodexにやらせている。リポジトリはcarina本体、aws、awscc、registry、github、infraの6つに分かれていて、それぞれのリポジトリで、複数のエージェントがworktreeを切って並行に作業する。

Rustのプロジェクトでこれをやると、worktreeの数だけ`target`ができて、そのどれもがビルドとテストを回す。ここ数日、PCが重い、ディスクが足りない、という状態になっていたので、やった対策をまとめておく。雑なメモです。

ちなみにマシンはMacBook Pro（Apple M4 Max、16コア（Performance 12、Efficiency 4）、メモリ128GB、SSD 2TB）。それなりに良いマシンのはずなんだけど、それでも重くなるし、ディスクも足りなくなる。

---

## ディスク容量

### ビルド出力専用のボリュームを作った

APFSのボリューム`/Volumes/CargoTargets`を作って、Time Machineの対象から外した。

以前は、ビルド出力を消しても空きがなかなか戻らなかった。消した分がTime Machineのローカルスナップショットに残っていたため。専用のボリュームにして対象から外したので、消せばすぐに空きが戻る。

### worktreeの`target`をそのボリュームへ向けた

`~/.local/bin/link-cargo-target`というスクリプトを書いて、worktreeの`target`を`/Volumes/CargoTargets/<repo>/<worktree>`へのシンボリックリンクにしている。メインのチェックアウトは対象外。

```sh
#!/bin/sh
# Point a linked worktree's `target/` at /Volumes/CargoTargets, a volume
# excluded from Time Machine, so deleted build output frees disk space
# immediately instead of lingering in local snapshots.
# Usage: link-cargo-target [worktree-dir]   (called from post-checkout hooks)
set -eu
vol=/Volumes/CargoTargets
wt=$(cd "${1:-.}" && git rev-parse --show-toplevel)
# Only linked worktrees; the main checkout keeps its own target/.
[ "$(git -C "$wt" rev-parse --git-dir)" != "$(git -C "$wt" rev-parse --git-common-dir)" ] || exit 0
[ -d "$vol" ] || exit 0
[ -e "$wt/target" ] || [ -L "$wt/target" ] && exit 0
repo=$(basename "$(dirname "$(git -C "$wt" rev-parse --path-format=absolute --git-common-dir)")")
dest="$vol/$repo/$(basename "$wt")"
mkdir -p "$dest"
ln -s "$dest" "$wt/target"
```

`--git-dir`と`--git-common-dir`が違えば、追加で作ったworktreeだと判断できる。メインのチェックアウトでは両方とも同じ`.git`を指すので、何もせずに抜ける。

infraを除くRustの5つのリポジトリで、post-checkoutフックからこれを呼ぶ。

```sh
#!/bin/sh
[ "$3" = 1 ] && ~/.local/bin/link-cargo-target . || true
```

post-checkoutフックの3番目の引数は、ブランチのチェックアウトなら`1`、ファイル単位のチェックアウトなら`0`になる。`git worktree add`でもこのフックが走るので、worktreeを作った時点でシンボリックリンクができる。

ひとつハマりどころがあって、`.gitignore`の`/target/`はシンボリックリンクには一致しない。末尾に`/`が付いたパターンはディレクトリにしか一致しないため。なので、5つのリポジトリすべてで`.git/info/exclude`に`/target`を足した。

### 消したworktreeのビルド出力を掃除する

worktreeを消しても、`/Volumes/CargoTargets`側のビルド出力は残る。なので、worktreeがもう存在しない`/Volumes/CargoTargets/<repo>/<worktree>`を消す`~/.local/bin/prune-cargo-targets`を書いた。

```sh
#!/bin/sh
# Delete per-worktree build dirs on /Volumes/CargoTargets whose worktree no longer exists.
# link-cargo-target creates <vol>/<repo>/<worktree>; removing the worktree leaves it behind.
set -eu
vol=/Volumes/CargoTargets
src=$HOME/src/github.com/carina-rs
[ -d "$vol" ] || exit 0
for d in "$vol"/*/*; do
  [ -d "$d" ] || continue
  repo=$(basename "$(dirname "$d")")
  wt=$(basename "$d")
  [ -e "$src/$repo/.worktrees/$wt" ] && continue
  echo "removing $d"
  rm -rf "$d"
done
```

worktreeはどのリポジトリでも`.worktrees/`の下に作る決まりにしているので、そこに同じ名前のディレクトリがなければ消す、というだけのもの。

これは、PRとCIを見張るスクリプト（`~/.local/bin/carina-watch`）から、30分ごとに呼ぶようにしている。初めて回したときは134GBが空いて、空きが206GiBから334GiBになった。

### Codexの書き込み先

Codexのサンドボックスから`/Volumes`以下に書き込むには、`--add-dir /Volumes/CargoTargets`を付ける必要がある。

---

## PCが重い

### Microsoft Defenderのシステム拡張を立ち上げ直した

いちばん効いたのがこれ。

```
sudo killall epsext
```

Defenderのリアルタイム保護はオフにしてあるのに、`epsext`が5日間、CPUを1コア分（約100%）使い続けていた。立ち上げ直したら、CPUは2.7%、メモリは2.9GBから16MBになった。

ついでにDefenderの除外に、CargoTargets、`~/Library/Caches/Mozilla.sccache`、`~/.cargo`、`~/.rustup`の4つを足してみたけど、これは効かなかった。リアルタイム保護がもともとオフなので、当然と言えば当然。

ただ、一連の対策を終えたあとで、リアルタイム保護はオンに戻した。なので、この除外は今はちゃんと意味を持っている（`~/src/`は前から除外済み）。

### CargoTargetsのSpotlight索引を止めた

```
sudo mdutil -i off /Volumes/CargoTargets
```

ビルド出力をSpotlightが索引に入れていたので、止めた。

---

## ビルドの負荷を抑える運用

ここからは、エージェントに守らせているルール。

- 1つのリポジトリで同時に動くエージェントは2つまで。これは負荷対策というより、並列にしすぎるとコンフリクトが起きるため。
- 手元でビルドするのは、マシン全体で同時に1つだけ。`nice -n 19`で回し、始める前に1分のload averageが32未満かを確かめる。
- workspace全体のテスト、clippy、doctestはCIに任せる。手元では、名前で選んだテストだけを流す。

---

## また重くなったときに見るもの

```
uptime
ps -Ao pcpu,rss,command -r | head
ps -o etime=,pcpu= -p $(pgrep -f epsext | head -1)
df -g /
```

- `epsext`がずっと高いなら、`sudo killall epsext`で立ち上げ直す。
- 空きが少ないなら、`prune-cargo-targets`を回す。それでも足りないなら、`tmutil listlocalsnapshots /`でローカルスナップショットを確かめる。
