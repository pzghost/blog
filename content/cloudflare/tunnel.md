---
title: "Tunnel"
isCJKLanguage: true
date: 2022-08-16T19:06:16Z
#lastmod: 2022-04-28T15:13:38+08:00
#categories: [CSI]
#tags: [kubernetes, csi, storage, volume]
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**
---
# [Connect through Cloudflare Access over SSH](https://developers.cloudflare.com/cloudflare-one/tutorials/ssh/)

- 通过命令行的方式
- 通过 Cloudflare 的 ui 方式(会自动创建一条 cmake 的 dns 记录)

```shell
# cat ~/.ssh/config
Host xxxxx
  ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
  User root
  IdentityFile ~/.ssh/vps

```
