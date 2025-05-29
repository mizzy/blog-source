---
title: Synology NASをTailscaleのExit Nodeにして外からLANにアクセス
date: 2025-05-29 09:04:51 +0900
---
外出先から自宅のLANに接続するために、Synology NASをExit Nodeとして設定した時のメモ。

Tailscaleのオフィシャルドキュメントに、Synology NAS上でのインストール/設定方法がまとめられているので、基本的にはこれを見ればよい。

[Access Synology NAS from anywhere · Tailscale Docs](https://tailscale.com/kb/1131/synology#manual-installation-steps)

ただしこのドキュメントでは、Exit Nodeとして設定してLANアクセスできるようにする方法は説明されていないので、こちらの動画も参考にした。

[Setup Your Synology NAS As A Tailscale Exit Node - YouTube](https://www.youtube.com/watch?v=F7mqVt_pUJY)

具体的には、Synology NASにSSHでログインして、以下のコマンドを実行すれば良い。

```shell
sudo tailscale up \
--advertise-exit-node \
--advertise-routes=192.168.100.0/24 \
--reset
```

その後、コンソール上でApproveする。

![](/images/2025/05/tailscale-exit-node.png)

これで外出先から、Tailscaleが入っていない/入れられない自宅LAN内のマシンにもアクセスできるようになった。
