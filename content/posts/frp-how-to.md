---
title: "如何使用 frp 搭建内网穿透服务方便地访问内网资源"
date: 2026-05-10T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - frp
  - frpc
  - frpc
  - 内网穿透
  - ssh tunnel
  - ssh 隧道
---

## 简介

本文简单介绍一下如何使用 frp 来做内网穿透和代理服务。

## 场景介绍

我们有以下机器：

1. 机器 A，位于一个局域网 A，例如校园网或者公司内网，没有公网 IP。
2. 机器 B，位于公网，有公网 IP，很容易访问到，可以通过 SSH 访问。
3. 机器 C，位于局域网 B，例如家里的局域网。

然后我们希望做到：

1. 从机器 C 可以访问到机器 A 以及与机器 A 在同一个局域网内的其他机器。
2. 机器 A 上直接部署代理服务，从机器 C 上可以直接使用机器 A 提供的代理服务。

由于机器 A 和机器 C 都可以通过 SSH 访问到机器 B，自然而然的，我们希望可以通过机器 B 作为中间桥梁来实现从机器 C 访问到机器 A，这个也就是所谓的内网穿透。

## 解决方案

要实现这种内网穿透，有很多工具和服务可以使用，本文主要考虑的是 [frp](https://github.com/fatedier/frp)。

frp 分为服务端 frps 和客户端 frpc。其中 frps 运行在机器 B 上，他负责中转。而客户端 frpc 运行在机器 A 上。当机器 C 连接到机器 B 的 frps 服务的时候，机器 B 的 frps 就负责机器 A 和机器 C 之间的双向通信。

## 详细部署

### 服务端

在机器 B 上，我们使用 docker compose 部署，`docker-compose.yaml` 如下：

```bash
name: frp
services:
  frps:
    image: snowdreamtech/frps:0.67.0
    restart: always
    environment:
      TZ: ${TZ:-Asia/Shanghai}
    ports:
      # frps 服务端口
      - "127.0.0.1:7000:7000"
      # web 页面端口，访问用户和密码见 frps.toml
      - "127.0.0.1:7500:7500"
      # 用于穿透的端口，这里我们使用6001-6010共计10个端口
      # 可以根据自己的实际需要增减
      # 这里内外的端口要保持一致，否则无法使用
      - "127.0.0.1:6001-6010:6001-6010"
    volumes:
      - ./frps.toml:/etc/frp/frps.toml:ro
      - ./volumes/logs:/var/log
```

其中 `frps.toml` 如下：

```toml
bindAddr = "0.0.0.0"
bindPort = 7000
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "your-webserver-password"
auth.method = "token"
auth.token = "your-auth-token"
log.level = "info"
# 每个代理服务的最大连接池大小
transport.maxPoolCount = 256
```

特别注意，这里我们把 frps 服务的端口都绑定到了 `127.0.0.1`，而非默认的 `0.0.0.0`，主要是因为我们想要使用 ssh tunnel 来连接到机器 B，避免直接把端口暴露到公网。如果你不介意暴露到公网，可以做对应的修改。

### 客户端

在机器 A 上，我们同样需要启动 frpc 的服务，但是由于机器 B 上的服务端口绑定到的是 127.0.0.1，我们需要先用 SSH 隧道来连接机器 B。为了方便，我们可以使用 autossh 来实现。修改 `~/.ssh/config` 配置如下：

```conf
Host my-tunnel
    # 这里写机器 B 的公网 IP
    HostName 1.1.1.1
    User ubuntu
    ServerAliveInterval 60
    ServerAliveCountMax 3
    # 端口转发配置
    # 这里特别设置本地端口和远程端口不一样，方便区分
    LocalForward 0.0.0.0:17000 127.0.0.1:7000
```

为了方便，我们可以写一个简单的 systemd unit 服务，创建并编辑文件 `~/.config/systemd/user/autossh-tunnel.service`：

```bash
[Unit]
Description=autossh for my tunnel
After=network.target

[Service]
Type=simple
KillMode=mixed
ExecStart=/usr/bin/autossh -M 0 -N my-tunnel

[Install]
WantedBy=default.target
```

然后启动服务 `systemctl --user start autossh-tunnel`。

接着是 frpc 的配置，这里同样使用 docker compose 来部署，`docker-compose.yaml` 如下：

```yaml
name: frp
services:
  frpc:
    image: snowdreamtech/frpc:0.67.0
    restart: always
    environment:
      TZ: ${TZ:-Asia/Shanghai}
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
```

其中 `frpc.toml` 如下：

```toml
# 这里写机器 A 的局域网 IP
serverAddr = "10.20.13.2"
serverPort = 17000
auth.method = "token"
auth.token = "your-auth-token"
log.level = "info"
# 连接池大小
transport.poolCount = 20

[[proxies]]
# socks5 代理服务
name = "socks5-proxy"
type = "tcp"
remotePort = 6001
# 开启加密和压缩
# 已经使用了 ssh tunnel，这里没有必要再加密，但是可选压缩
#transport.useEncryption = true
#transport.useCompression = true

[proxies.plugin]
type = "socks5"

[[proxies]]
# http 代理服务
name = "http-proxy-proxy"
type = "tcp"
remotePort = 6002
# 开启加密和压缩
# 已经使用了 ssh tunnel，这里没有必要再加密，但是可选压缩
#transport.useEncryption = true
#transport.useCompression = true

[proxies.plugin]
type = "http_proxy"
```

### 访问端

在机器 C 上，我们同样需要创建一个到机器 B 的 SSH 隧道，然后就可以使用内网穿透了。修改 `~/.ssh/config` 配置如下：

```conf
Host my-tunnel
    # 这里写机器 B 的公网 IP
    HostName 1.1.1.1
    User ubuntu
    ServerAliveInterval 60
    ServerAliveCountMax 3
    # 端口转发配置
    # 这里特别设置本地端口和远程端口不一样，方便区分
    # 这里我们不需要转发 7000，因为我们这里不用 frpc，只需要转发对应的代理服务的端口即可
    LocalForward 0.0.0.0:16001 127.0.0.1:6001
    LocalForward 0.0.0.0:16002 127.0.0.1:6002
```

然后同样的 为了方便，我们可以写一个简单的 systemd unit 服务，创建并编辑文件 `~/.config/systemd/user/autossh-tunnel.service`：

```bash
[Unit]
Description=autossh for my tunnel
After=network.target

[Service]
Type=simple
KillMode=mixed
ExecStart=/usr/bin/autossh -M 0 -N my-tunnel

[Install]
WantedBy=default.target
```

然后启动服务 `systemctl --user start autossh-tunnel`。

最后，在机器 C 上，我们就可以直接使用 `socks5://127.0.0.1:16001` 或者 `http://127.0.0.1:16002` 去访问到机器 A 及其所在局域网的资源了。

## 总结

本文简单介绍了如何使用 frp 来实现内网穿透，包括如何使用 frp 自带的插件来在内网中启动一个 socks5 或者 http 代理服务，并透过 SSH 隧道提升安全性。
