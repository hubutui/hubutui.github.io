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
  - 明御运维审计与风险控制系统
  - 安恒
  - 堡垒机
  - 跳板机
---

## 简介

本文简单记录如何使用 Docker 容器来登录 VPN，并对外提供 HTTP 和 SOCKS5 代理，以方便本地机器使用的同时，不因为 VPN 导致我们的电脑本身的网络访问其他网络出现问题。

本文主要针对 Linux 系统，macOS 和 Windows 也可以参考使用，对于使用 docker 的部分，实际上由于我们主要使用它来提供代理，也可以在 Linux 机器上部署和运行容器，然后给其他的机器（不限操作系统）使用。

## 快速说明

简单地说，我们其实是把 VPN 软件安装到了 Docker 容器内，并对外提供 HTTP 和 SOCKS5 代理，然后在容器外部直接设置需要这个 VPN 的程序使用这个代理去访问，从而避免了 VPN 对我们本机网络的干扰。

除此之外，我们还提供简单的 SSH 配置，方便我们可以直接在终端直接 ssh 登录，方便使用。

由于我们提供来代理，实际上这个代理是可以给多个用户使用的。

本文说明的内容主要针对深信服 easyconnect 和 atrust，华为 UniVPN，堡垒机主要是安恒的明御堡垒机。如果是其他 VPN，也可以参考使用。

## 需要提前准备的软件

1. docker，最好是搭配 docker compose 使用，否则请自行把 docker compose 配置转换为对应的 docker 命令。这个是必需的。
2. chrome 之类的浏览器，搭配一个代理管理插件，如 ZeroOmega，用于浏览器登录机和代理服务器配置。这个是必需的。
3. VNC 客户端，如 [TigerVNC](https://tigervnc.org), [RealVNC](https://www.realvnc.com)，用于连接 Docker 容器内的简单桌面，进行 VPN 的登录。这个是必需的。注意，我们只需要 VNC 客户端，不需要它的服务端，下载安装的时候需要注意选择。
4. nc 命令，本文使用的是 [openbsd-netcat](https://salsa.debian.org/debian/netcat-openbsd)，如使用其他实现，请注意命令选项是否相同。`nc` 命令用于配置 SSH 的代理命名。这个是必需的。
5. RDP 客户端，如 [remmina](https://remmina.org) 和 [xfreerdp](https://www.freerdp.com)，用于通过堡垒机连接 Windows 机器。这个是根据情况可选的。
6. [proxychains-ng](https://github.com/rofl0r/proxychains-ng) 之类的代理工具，用于启动本地堡垒机客户端的时候自动套一层代理。这个是根据情况可选的。
7. [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty)，用于通过 SSH 连接 Linux 服务器。不过这里更加建议使用 Linux 或者 macOS 的终端直接直接使用 ssh 命令。

## 深信服 easyconnect VPN

这里我们使用 [docker-easyconnect](https://github.com/docker-easyconnect/docker-easyconnect) Docker 镜像来提供 socks5 和 HTTP 代理。

docker compose 配置示例如下：

```yaml
services:
  easyconnect:
    image: hagb/docker-easyconnect:7.6.7
    restart: unless-stopped
    devices:
      - "/dev/net/tun:/dev/net/tun"
    cap_add:
      - NET_ADMIN
    environment:
      # VNC 密码
      - PASSWORD=123456
      - URLWIN=1
      # 启用 noVNC
      - USE_NOVNC=1
    volumes:
      - ./volumes/easyconnect:/root
    ports:
      # VNC 端口
      - "127.0.0.1:5901:5901"
      # noVNC 端口
      - "127.0.0.1:8080:8080"
      # HTTP 代理端口
      - "127.0.0.1:8888:8888"
      # SOCKS5 代理端口
      - "127.0.0.1:1080:1080"
```

然后使用 VNC 客户端连接到 `127.0.0.1:5901`，VNC 的密码为命令行中设置的 `<VNC_PASSWD>`。没有 VNC 客户端的也可以通过浏览器访问 `127.0.0.1:8080`，使用 VPC 密码登录。接着在打开的 easyconnect 图形界面中输入 VPN 服务器地址、用户名、密码、验证码等信息即可登录。VPN 登录成功之后直接关闭 VNC 客户端即可。

这里端口映射的 `127.0.0.1` 表示只允许本机访问，如果允许局域网内的其他机器也来使用，可以去掉 `127.0.0.1`。一般认为直接把代理端口绑定到 `0.0.0.0` 是不安全的。

这样，我们就配置好 easyconnect，在 `127.0.0.1:1080` 和 `127.0.0.1:8888` 上分别提供 SOCKS5 和 HTTP 代理了。一般我们优先使用 SOCKS5 代理。

## 深信服 atrust VPN

这里与 easyconnect 的方法类似，不过我们这里可以尝试使用一个支持自动重新登录的镜像，它也是基于 [docker-easyconnect](https://github.com/docker-easyconnect/docker-easyconnect) 改进而来的。

如果想要自动跳过图形验证码和自动登录，可以参考 [aTrustLogin](https://github.com/kenvix/aTrustLogin)。在浏览器上先登录一次，打开浏览器的开发者工具->Application->Cookies，找到 `tid` 和 `tid.sig` 的值，填入下面的 docker compose 配置中。
不需要自动处理图形验证码的，省略 `--cookie_sig` 和 `--cookie_tid` 参数，VPN 连接到容器之后手动登录即可。
环境变量 `PING_ADDR` 用于让容器自动 ping 该地址以保活避免连接断开。自动登录和保活不一定能够生效。

```yaml
services:
  atrust:
    image: kenvix/docker-atrust-autologin:latest
    environment:
      - ATRUST_OPTS=--portal_address="https://114.114.114.114" --username=USERNAME --password='PASSWORD' --cookie_sig COOKIE_SIG --cookie_tid COOKIE_TID
      - PASSWORD=123456
      - URLWIN=1
      - PING_ADDR=10.10.10.10
    volumes:
      - ./volumes/atrust:/root
    ports:
      # VNC 端口
      - "127.0.0.1:5901:5901"
      # noVNC 端口
      - "127.0.0.1:8080:8080"
      # HTTP 代理端口
      - "127.0.0.1:8888:8888"
      # SOCKS5 代理端口
      - "127.0.0.1:1080:1080"
    cap_add:
      - NET_ADMIN
    devices:
      - "/dev/net/tun"
    sysctls:
      - net.ipv4.conf.default.route_localnet=1
    shm_size: 1G
    restart: unless-stopped
```

深信服 atrust 应该是新的 VPN 登录方式，如果支持的话，建议先用这个。

## 华为 UniVPN

华为 VPN 早先是用的 SecoClient，但是这个已经不再支持了，目前推荐使用的是联软科技开发的 UniVPN。而且已有的需要 SecoClient 登录的 VPN 一般也可以用 UniVPN 来登录。可以参考[官方公告](https://support.huawei.com/enterprise/zh/bulletins-website/ENEWS2000014193)。

这个主要参考 [docker-univpn](https://github.com/zx900930/docker-univpn)。

docker compose 配置如下：

```yaml
services:
  univpn:
    image: triatk/univpn:10781.18.1.512
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - "/dev/net/tun:/dev/net/tun"
    # 如果 VPN 有要求绑定特定的 MAC 地址，则在此设置
    mac_address: ${SPOOF_MAC:-02:42:ac:11:00:01}
    ports:
      # VNC 端口
      - "127.0.0.1:5901:5901"
      # noVNC 端口，可以使用 http://宿主机IP:6901 访问
      - "127.0.0.1:6901:6901"
      # socks5 代理接口
      - "127.0.0.1:1080:1080"
      # http 代理接口
      - "127.0.0.1:8888:8888"
    shm_size: 2G
    environment:
      VNC_PW: ${VNC_PASSWORD:-univpn}
      TZ: ${TZ:-Asia/Shanghai}
      VNC_DEPTH: ${VNC_DEPTH:-24}
      VNC_RESOLUTION: ${VNC_RESOLUTION:-1280x1024}
    volumes:
      - ./univpn_config:/home/vpnuser/UniVPN
      - ./univpn_logs:/usr/local/UniVPN/log
```

## 堡垒机登录

有的时候，登录了 VPN 之后还不能直接访问到目标服务器，还需要经过堡垒机。一般常用的堡垒机是安恒的明御运维审计与风险控制系统（堡垒机）。

### 浏览器使用

堡垒机一般可以直接使用浏览器登录，只需要设置浏览器的代理地址为我们前面配置好的 docker 容器提供的代理服务器地址即可。chrome 系列浏览器推荐使用 [ZeroOmega](https://github.com/zero-peak/ZeroOmega) 之类的插件，然后添加代理服务器和自动切换的配置，设置指定堡垒机的服务器地址自动使用我们设置的代理服务器，其他流量直接连接，即可方便地分流正常的流量和需要 VPN 的流量。

直接使用浏览器访问堡垒机并且在浏览器内进行操作是比较简单的，推荐使用。

### 运维工具客户端

明御堡垒机有提供客户端给用户下载安装后使用，但是实际上就是一个 electron 封装的而已，考虑到它还不方便设置代理服务器，还不如浏览器方便使用。

Linux 用户可以下载安装包之后解压使用，这个时候需要使用 [proxychains-ng](https://github.com/rofl0r/proxychains-ng) 套一层来使用代理。但是他也有自己的优点：

1. 可以在客户端里配置一下 RDP，让它启动本地的 xfreerdp 或者 remmina 来连接需要通过堡垒机访问的 Windows 桌面。
2. 在客户端里配置一下 SSH 登录，让他启动本地的 PuTTY 来连接需要通过堡垒机登录的 Linux 服务器。不过我们一般更加建议直接修改 OpenSSH 配置文件来方便使用 ssh 命令登录。

在浏览器这里是没法做这个操作的，因为浏览器启动本地的其他程序的时候没法设置代理服务器，导致无法连接。

### OpenSSH 配置

堡垒机上有个 OpenSSH 配置让我们下载并且安装，方便我们可以直接使用 ssh 命令从本地机器直接连接到目标 Linux 服务器。但是这个配置其实不太好用。建议换一种方式配置。

1. 把你本地机器的 SSH 公钥添加到堡垒机里，一般在堡垒机页面的个人信息页面里添加。
2. 修改 `~/.ssh/config` 配置如下：

```conf
Host usm
  # 堡垒机地址
  HostName 10.10.10.11
  User username
  # 添加到堡垒机的 SSH 公钥对应的私钥文件
  IdentityFile ~/.ssh/id_ed25519
  # 这个端口一般都是这个，不是的话得问管理员
  Port 60022
  # 配置代理连接命令
  ProxyCommand nc -X 5 -x 127.0.0.1:1080 %h %p

Host server01
  HostName 172.17.10.10
  User user
  Port 22
  ProxyJump usm
```

说明：

1. 这里的 nc 命令由 openbsd-netcat 提供，如果你用的是其他实现的 nc，需要检查这个命令的参数是否设置正确。
2. 上面这个 `usm` 配置表示，我们 ssh 连接到堡垒机的时候，使用的 SSH 私钥和代理。直接输入 `ssh usm` 应该就可以登录到堡垒机了。登录结果应该就是看到一个交互式的菜单，我们可以在此选择某个 Linux 主机去登录，然后输入用户名和密码。
3. 部分堡垒机还支持直接做跳转，也就是上面配置的 `server01`，它表示 `ssh server01` 的时候会使用 `usm` 作为跳板机去登录，而跳板机 `usm` 要通过代理服务器去登录。
4. 最终实现的效果就是，`ssh server01` 即可自动地通过 VPN 提供的代理服务器去登录 `usm`，再从 `usm` 跳转到目标服务器 `server01`。
5. 如果管理员配置允许，我们还能直接使用 `rsync` 和 `scp` 命令从本地传输文件到 `server01`，不过一般用到堡垒机的场景恐怕都不太允许这些操作。
6. 甚至有的堡垒机配置都不允许使用 `ssh server01` 自动地从 `usm` 跳转登录到目标服务器，只能先 `ssh usm` 登录到 `usm` 之后再手动登录。
7. 有的堡垒机出于安全要求，还要求做两步验证，一般就是在登录堡垒机的时候输入手机验证码。如果可以，建议查看堡垒机页面上的个人设置，看看是否可以添加身份验证器。可以的话，建议使用 Microsoft Authenticator 之类的身份验证器，这样还能避免手机验证码接收延迟太长导致无法登录的问题。

## 总结

本文简单介绍了如何在 Docker 容器内运行 VPN 软件，并对外提供一个代理，以方便宿主机使用，并补充提供了方便直接使用代理和跳板机登录的 OpenSSH 配置。
