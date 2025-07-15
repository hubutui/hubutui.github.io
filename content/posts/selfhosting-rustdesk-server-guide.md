---
title: "如何配置自托管的 rustdesk 服务并指定自定义的端口"
date: 2025-07-15T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - rustdesk
  - docker
  - remote desktop
  - vnc
---

## 前言

[rustdesk](https://github.com/rustdesk/rustdesk) 是一个以 AGPL 协议开源的自由软件，主要是用于远程桌面控制，类似 ToDesk, TeamViewer 等．

使用 rustdesk 需要有一个服务器端作为中介，客户端通过这个中介进行通信．官方的服务器在国外，访问不太方便．我们可以自己搭建一个服务器端来为我们自己提供服务．

本文主要介绍的是使用 docker compose 来启动服务，并且指定不同的端口．

## 服务端介绍

rustdesk 服务端配置，主要包含两个服务：

1. ID 服务 hbbs
2. 中继服务 hbbr

## 端口

rustdesk 服务端需要以下端口：

- hbbs:
  - 21114 (TCP): 用于网页控制台，仅在 Pro 版本中可用．
  - 21115 (TCP): 用于 NAT 类型测试．
  - 21116 (TCP/UDP): 请注意 21116 应该同时为 TCP 和 UDP 启用． 21116/UDP 用于 ID 注册和心跳服务．21116/TCP 用于 TCP 打洞和连接服务．
  - 21118 (TCP): 用于支持网页客户端．
- hbbr:
  - 21117 (TCP): 用于中继服务．
  - 21119 (TCP): 用于支持网页客户端．

其中 21118 和 21119 不需要可以不开放．

但是用 docker 部署的时候，这些端口可能被占用了．此时，最好是让容器内部和宿主机端口保持一致．通过指定服务的 PORT 环境变量来修改默认的端口．

具体的，我们的 `docker-compsoe.yaml` 文件如下：

```yaml
services:
  hbbs:
    image: rustdesk/rustdesk-server:1.1.14
    command: hbbs
    volumes:
      - ./data:/root
    ports:
      - 2035:2035
      - 2036:2036/tcp
      - 2036:2036/udp
      - 2038:2038
    environment:
      # 对应于 21116 的端口
      PORT: 2036
    depends_on:
      - hbbr
    restart: always

  hbbr:
    image: rustdesk/rustdesk-server:1.1.14
    command: hbbr
    volumes:
      - ./data:/root
    ports:
      # 对应于 21117 的端口
      - 2037:2037
      - 2039:2039
    environment:
      PORT: 2037
    restart: always
```

## 客户端配置

在设置->网络中，ID 服务器写入：`IP:2036`，中继服务器写入：`IP:2037`，key 写入文件 [data/id_ed25519.pub](data/id_ed25519.pub) 中的内容即可．这里 IP 指的是部署 rustdesk 服务端的公网服务器的公网 IP．

## 参考

1. 官方文档：[自建服务器开源版](https://rustdesk.com/docs/zh-cn/self-host/rustdesk-server-oss)
