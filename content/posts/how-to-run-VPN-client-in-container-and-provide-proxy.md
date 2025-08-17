---
title: "如何在 docker 容器中运行 VPN 客户端并提供代理服务"
date: 2025-08-17T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - docker
  - VPN
  - anyconnect
  - atrust
  - 深信服
  - secoclient
  - 华为
  - UniVPN
---

## 前言

在 Linux 系统下，我们有的时候需要通过一些 VPN 客户端登录到某个网络里．这些 VPN 有深信服、华为等等，但是我们不想污染我们的系统，此时可以考虑使用 docker 来解决．让这些 VPN 客户端运行在容器内，然后通过容器对外提供 HTTP 和 socks5 代理，方便自己使用，同时这个代理还可以提供局域网的其他用户直接使用．

## 正文

这里不废话，直接给出可以使用的解决方案．

1. 深信服 easyconnect，直接参考 [docker-easyconnect](https://github.com/docker-easyconnect/docker-easyconnect) 这个项目即可．
2. 深信服 atrust，可以参考 [docker-easyconnect](https://github.com/docker-easyconnect/docker-easyconnect)，他同时也支持 atrust．此外，还可以考虑基于这个项目的另外一个项目 [aTrustLogin](https://github.com/kenvix/aTrustLogin)，它额外支持了自动登录．
3. 华为 secoclient，或者 UniVPN，实际上 secoclient 已经不再维护，如果可以建议优先考虑使用它的后继者 UniVPN．可以参考 [docker-univpn](https://github.com/zx900930/docker-univpn)．
