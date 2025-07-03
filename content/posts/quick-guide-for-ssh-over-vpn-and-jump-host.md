---
title: "透过 VPN 和堡垒机使用 SSH 登录目标服务器的快速指南"
date: 2025-07-03T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - ssh
  - vpn
  - proxy server
  - proxy jump
  - netcat
---

## 前言

考虑这样一种情况，我们需要使用 VPN 登录到内网，然后还需要经过一个堡垒机，堡垒机提供网页端服务，在上面点击可以 SSH 登录到目标服务器。但是在网页上使用的命令行终端不太方便使用，传输文件也麻烦。如果能够直接通过命令行登录，则会非常简单好用。实际上这个是可以做到的。

## SSH 配置

一般的，我们为了方便，可以编辑 `~/.ssh/config` 文件，例如：

```text
Host server1
  HostName 192.168.1.101
  User username
  Port 6022
```

这样，我们可以直接使用 `ssh server1` 命令即可登录，`rsync` 和 `scp` 命令也可以直接用 `server1` 这个名字。

实际上，这个配置是支持设置代理和堡垒机的。假设我们使用的是深信服的 easyconnect 或者 atrust，我们可以使用 [docker-easyconnect](https://github.com/docker-easyconnect/docker-easyconnect) 来实现登录并提供一个 socks5 和 http 代理。假设我们配置好了 socks5 代理在本地的 `1081` 端口，我们可以这样写：

```bash
Host server1
  HostName 192.168.1.101
  User username
  Port 6022
  # netcat from openbsd-netcat
  ProxyCommand netcat -X 5 -x 127.0.0.1:1081 %h %p
  # ncat from nmap
  # ProxyCommand ncat --proxy-type socks5 --proxy 127.0.0.1:1081 %h %p
```

这里的代理命令可以用 [openbsd-netcat](https://salsa.debian.org/debian/netcat-openbsd) 提供的 `netcat`，也可以用 [nmap](https://nmap.org) 提供的 `ncat`。前者相对比较简单一些。

如果你的服务器只需要通过 VPN 即可访问，这样配置即可。如果还要求使用堡垒机，则还需要更加复杂的配置：

```bash
Host usm
  HostName 192.168.20.11
  # 堡垒机的登录用户名
  User username
  # 堡垒机的默认端口，常见的为 60022
  Port 60022
  # netcat from openbsd-netcat
  ProxyCommand netcat -X 5 -x 127.0.0.1:1081 %h %p
  # ncat from nmap
  # ProxyCommand ncat --proxy-type socks5 --proxy 127.0.0.1:1081 %h %p

Host server1
  HostName 192.168.1.101
  User username
  Port 6022
  ProxyJump usm
```

一般的，我们需要在堡垒机的设置中添加你的 ssh 公钥，具体设置方法根据不同的堡垒机厂商有所不同，请根据需要查看对应文档。

然后，我们即可使用 `ssh server1` 直接连接到服务器了。这个连接首先会经过我们配置的 socks5 代理，然后登录到堡垒机 `usm`，再从堡垒机最终登录到 `server1`，只是这一切都自动处理了。`rsync` 和 `scp` 命令也可以正常使用。

不过，一般来说，如果想要创建 SSH 隧道，堡垒机可能会拦截，一般是不允许的。

## 补充

以上配置都是针对 Linux 下的，直接编辑修改 `~/.ssh/config` 即可。如果是使用其他的 SSH 客户端，例如 XShell, MobaXTerm，则根据以上信息设置对应的代理服务器和堡垒机即可。
