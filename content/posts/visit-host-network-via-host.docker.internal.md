---
title: "在容器内使用 host.docker.internal 访问宿主机"
date: 2026-03-22T00:00:00+08:00
toc: false
images:
tags:
  - docker
  - 容器
  - podman
  - 网络配置
---

## 前言

Windows 和 OS X 的 docker 容器一般用的 docker desktop 管理，一般来说实际上是在一个 Linux 虚拟机内运行的容器。为了方便，docker desktop 自动给容器注入了特殊的 DNS 解析，可以让容器内通过 `host.docker.internal` 这个域名解析到宿主机，从而方便容器访问宿主机的网络。

## Linux 下的情况

但是，在 Linux 下，docker 容器里并没有添加这个特殊的 DNS 解析，因此无法直接使用 `host.docker.internal` 来访问宿主机。不过其实也可以直接用宿主机的 IP 来访问。

此外，除了 docker，一般我们更加常用 podman 来运行容器。而 podman 为了方便，实际上给我们增加了 `host.docker.internal` 和 `host.containers.internal`，方便用户在容器内使用这两个域名来访问到宿主机网络。

例如，在一个 podman 容器内，我们可以查看 `/etc/hosts` 的内容如下：

```
169.254.1.2     host.docker.internal
169.254.1.2     host.containers.internal
127.0.0.1       localhost
::1     localhost
172.16.6.2      17913cf8e0bd test-server01-1
```

这里可以看到 `host.containers.internal` 和 `host.docker.internal` 都被解析为 `169.254.1.2`，也就是你的宿主机在容器网络里的 IP。

特别注意，你在宿主机上的网络服务应该要绑定到 `0.0.0.0` 才能够让容器内的请求到达，绑定到 `127.0.0.1` 的是不行的。

## docker compose 配置

为了统一处理，在 docker compose 中，我们可以自己添加 `extra_hosts`，无论是 docker 还是 podman 都可以正常使用。配置例子如下：

```yaml
name: test
services:
  server01:
    image: ubuntu:22.04
    entrypoint: ["tail", "-f", "/dev/null"]
    extra_hosts:
      - "host.docker.internal:host-gateway"
      - "host.containers.internal:host-gateway"
```

这里的 `host-gateway` 是一个特殊的占位符，他会被 docker / podman 自动替换为对应的 IP。
