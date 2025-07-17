---
title: "如何使用 podman 替代 docker"
date: 2025-07-17T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - docker
  - podman
  - docker compose
  - podman compose
  - podman-compose
  - rootless docker
  - rootless podman
  - rootless container
---

## 前言

早就听闻 podman 是 docker 的替代者，而且 podman 默认不需要用 root 用户来运行一个类似 dockerd 的守护进程，更加安全，此外 podman 的命令也基本与 docker 的命令保持一致，迁移过来也十分简单．

本文将以使用者的角度，简单聊聊如何使用 podman 替代 docker．

没有特别说明的话，本文使用 Ubuntu 24.04 操作系统来演示 podman 的使用．

## 安装 podman 以及简单使用

安装 podman 其实非常简单，直接从官方源安装即可：

```bash
sudo apt install -y podman
```

安装完成之后即可直接使用 podman 命令了，他的用法与 docker 命令基本一样．你甚至可以安装一个包 `podman-docker`，从而让你继续使用 docker 命令，但是实际上是操作 podman 容器．注意，这里的 podman-docker 包与 docker 包是冲突的，所以安装这个包就不能保留 docker 包了．

podman 默认不需要一个类似 dockerd 的守护进程，而且默认是可以以非 root 身份管理和运行容器的，他也不需要加入一个类似 docker 组的特殊权限组．

一般的，我们建议直接以普通用户的身份来执行 podman 命令．

## podman 的配置

为了让 podman 用起来更像 docker 那样，我们可以配置一下 podman 的镜像仓库，创建并编辑 `/etc/containers/registries.conf.d/10-unqualified-search-registries.conf`，内容如下：

```conf
unqualified-search-registries = ["docker.io"]
```

## podman/docker 的快速入门介绍

![docker](https://github.com/hubutui/docker-for-env-without-internet-access/raw/master/.images/docker.jpg)

如图所示，我们可以以 Docker 镜像为中心快速入门 docker 的使用．当然，这个对于 podman 也是同样适用的．

1. 我们使用 `docker run` 命令来从一个 docker 镜像创建出来一个容器，然后在容器内进行各种操作．一般建议使用 `docker run -it --rm`，这样当你退出容器的时候就会自动删除了．实际生产环境很少直接使用 `docker run` 命令了，一般建议使用 docker compose 来管理．
2. `docker commit` 可以把正在运行的容器保存为镜像，但是我们一般不建议使用，因为它保存的镜像文件往往太大，也无法重复．
3. 最佳实践是从 `Dockerfile` 使用 `docker build` 命令构建镜像．关于 Dockerfile 的编写，请参考这个[文档](https://yeasy.gitbook.io/docker_practice/image/dockerfile)，主要掌握 `FROM, COPY, CMD, ENV, ARG, WORKDIR` 这几个指令，其他指令可以暂时忽略．
4. 从仓库拉取镜像可以使用 `docker pull` 命令．一般我们不需要推送自己的镜像到仓库去，如果有需要可以查阅 `docker push` 命令和对应仓库站点的说明（比如如何注册和登录）．
5. 更多的时候我们可能需要在机器 A 导出镜像文件，然后导入到机器 B．导出镜像为文件可以使用命令 `docker save`，一般建议使用压缩，例如 `docker save nginx:latest | zstd -o nginx.tar.zst`．导出只需要使用 `docker load -i nginx.tar.zst` 即可．除非你用了很老版本的操作系统和 docker 版本，否则导入的时候是自动支持解压的，我们不需要额外的处理．

本小结内容整理自 [docker-for-env-without-internet-access](https://github.com/hubutui/docker-for-env-without-internet-access)，如有需要也可以参考．

## 一些扩展

### 使用 docker 命令（非 podman-docker）来管理 podman 容器

实际上 podman 有一个 systemd 服务可以启动的，只是 podman 本身是不需要它的．这个服务提供兼容 docker 1.40 API 的层，也就是说启动这个服务之后，可以直接使用原生的 docker 命令来操作 podman 容器，就好像使用 docker 命令跟 dockerd 守护进程通信一样．

```bash
sudo systemctl stop docker.socket
sudo systemctl stop docker.service
# 这里我们使用普通用户 ubuntu
sudo loginctl enable-linger ubuntu
# 这里我们直接启动 podman.socket
# 也可以启动 podman.service，差别在于前者只会在有请求的时候才会去真正的启动 podman.socket
systemctl --user start podman.socket
systemctl --user enable podman.socket
# 设置 DOCKER_HOST 为 podman.socket，让 docker 跟这个套接字通信
# 这个配置也可以写到 .bashrc 里去
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
```

这样，我们就可以直接使用 docker 命令来操作 podman 容器了．例如，`docker info` 输出：

```text
host:
  arch: amd64
  buildahVersion: 1.33.7
  cgroupControllers:
  - cpu
  - memory
  - pids
  cgroupManager: systemd
  cgroupVersion: v2
  conmon:
    package: conmon_2.1.10+ds1-1build2_amd64
    path: /usr/bin/conmon
    version: 'conmon version 2.1.10, commit: unknown'
  cpuUtilization:
    idlePercent: 99.88
    systemPercent: 0.04
    userPercent: 0.07
  cpus: 2
  databaseBackend: sqlite
  distribution:
    codename: noble
    distribution: ubuntu
    version: "24.04"
  eventLogger: journald
  freeLocks: 2046
  hostname: aws.butui.me
  idMappings:
    gidmap:
    - container_id: 0
      host_id: 1000
      size: 1
    - container_id: 1
      host_id: 100000
      size: 65536
    uidmap:
    - container_id: 0
      host_id: 1000
      size: 1
    - container_id: 1
      host_id: 100000
      size: 65536
  kernel: 6.8.0-1031-aws
  linkmode: dynamic
  logDriver: journald
  memFree: 118321152
  memTotal: 958533632
  networkBackend: netavark
  networkBackendInfo:
    backend: netavark
    dns:
      package: aardvark-dns_1.4.0-5_amd64
      path: /usr/lib/podman/aardvark-dns
      version: aardvark-dns 1.4.0
    package: netavark_1.4.0-4_amd64
    path: /usr/lib/podman/netavark
    version: netavark 1.4.0
  ociRuntime:
    name: crun
    package: crun_1.14.1-1_amd64
    path: /usr/bin/crun
    version: |-
      crun version 1.14.1
      commit: de537a7965bfbe9992e2cfae0baeb56a08128171
      rundir: /run/user/1000/crun
      spec: 1.0.0
      +SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +WASM:wasmedge +YAJL
  os: linux
  pasta:
    executable: /usr/bin/pasta
    package: passt_0.0~git20240220.1e6f92b-1_amd64
    version: |
      pasta unknown version
      Copyright Red Hat
      GNU General Public License, version 2 or later
        <https://www.gnu.org/licenses/old-licenses/gpl-2.0.html>
      This is free software: you are free to change and redistribute it.
      There is NO WARRANTY, to the extent permitted by law.
  remoteSocket:
    exists: true
    path: /run/user/1000/podman/podman.sock
  security:
    apparmorEnabled: false
    capabilities: CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_FOWNER,CAP_FSETID,CAP_KILL,CAP_NET_BIND_SERVICE,CAP_SETFCAP,CAP_SETGID,CAP_SETPCAP,CAP_SETUID,CAP_SYS_CHROOT
    rootless: true
    seccompEnabled: true
    seccompProfilePath: /usr/share/containers/seccomp.json
    selinuxEnabled: false
  serviceIsRemote: false
  slirp4netns:
    executable: /usr/bin/slirp4netns
    package: slirp4netns_1.2.1-1build2_amd64
    version: |-
      slirp4netns version 1.2.1
      commit: 09e31e92fa3d2a1d3ca261adaeb012c8d75a8194
      libslirp: 4.7.0
      SLIRP_CONFIG_VERSION_MAX: 4
      libseccomp: 2.5.5
  swapFree: 0
  swapTotal: 0
  uptime: 45h 7m 1.00s (Approximately 1.88 days)
  variant: ""
plugins:
  authorization: null
  log:
  - k8s-file
  - none
  - passthrough
  - journald
  network:
  - bridge
  - macvlan
  - ipvlan
  volume:
  - local
registries:
  search:
  - docker.io
store:
  configFile: /home/ubuntu/.config/containers/storage.conf
  containerStore:
    number: 2
    paused: 0
    running: 2
    stopped: 0
  graphDriverName: overlay
  graphOptions: {}
  graphRoot: /home/ubuntu/.local/share/containers/storage
  graphRootAllocated: 19682557952
  graphRootUsed: 3230658560
  graphStatus:
    Backing Filesystem: extfs
    Native Overlay Diff: "true"
    Supports d_type: "true"
    Supports shifting: "false"
    Supports volatile: "true"
    Using metacopy: "false"
  imageCopyTmpDir: /var/tmp
  imageStore:
    number: 2
  runRoot: /run/user/1000/containers
  transientStore: false
  volumePath: /home/ubuntu/.local/share/containers/storage/volumes
version:
  APIVersion: 4.9.3
  Built: 0
  BuiltTime: Thu Jan  1 08:00:00 1970
  GitCommit: ""
  GoVersion: go1.22.2
  Os: linux
  OsArch: linux/amd64
  Version: 4.9.3
```

明显看到，这些都是 podman 的信息．

### docker-compose 和 podman-compose

直接使用 docker 或者 podman 命令过于简陋，上 k3s 或者 k8s 又过于复杂．这个时候 docker-compose 或者 podman-compose 就是一个比较好的折中选择，它使用一个简单的 `docker-compose.yaml` 文件来描述启动一个项目需要启动的服务（容器），然后只需要一个 `docker compose up -d` 或者 `podman compose up -d` 即可启动服务，非常简单好用．

`podman compose` 实际上可以使用 [podman-compose](https://github.com/containers/podman-compose) 也可以使用 [docker-compose](https://docs.docker.com/compose)．如果后者已经安装了，默认它会优先使用 docker-compose．特别注意的是，docker-compose 有 v1 和 v2 两个差别较大的版本，建议使用 v2．

此外，docker-compose 实际上还是需要跟 dockerd 通信，因此要让 podman 能够正常使用 docker-compose 的话，需要使用 podman.socket 的．

你可以可以参考[文档](https://docs.podman.io/en/latest/markdown/podman-compose.1.html)来设置让 podman 默认使用哪一个．

一般认为 podman-compose 对 docker compose 的兼容性有一些小问题，例如环境变量的设置，见 [issue #491](https://github.com/containers/podman-compose/issues/491)．建议优先选择 docker-compose．

当然了，实际上你也可以直接使用 `docker compose` 命令的．

## 小结

本文简单介绍了如何使用 podman 替代 docker，还给出了 podman compose 的相关用法，以及如何使用 docker 命令来操作 podman 容器等．

1. 仅安装 podman 和 podman-compose，不安装 docker，直接使用 podman 替代 docker 命令，基本的命令以及 podman compose 都是可以正常使用的．
2. 安装 podman 和 docker-compose-v2（docker 可能会被作为依赖安装），启动 `podman.socket/service`，禁用和停止 `docker.socket/service`，配置环境变量 `DOCKER_HOST`，正常使用 docker 命令，实际操作的是 podman 容器．当然，也可以用 podman 命令．
