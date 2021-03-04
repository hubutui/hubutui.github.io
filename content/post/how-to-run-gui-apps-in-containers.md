---
title: "在容器中运行 GUI 程序的几种方法"
date: 2021-03-04T00:00:00+08:00
---

首先，我们常用的容器有这么几种：

* 最为流行也最广为人知的 [Docker](https://www.docker.com)
* 几个大厂联合推出来想要替代 Docker 的 [Podman](https://podman.io)
* 适用于 HPC 场景的 [Singularity](https://sylabs.io/singularity)
* Ubuntu 曾经推的 [LXD](https://linuxcontainers.org/lxd/introduction)

其中，LXD 与其他几个容器不太一样，是可以当成一个虚拟机来用的系统容器，里面安装和使用 systemd，SSH，GUI 程序，都没有啥大问题。这里就不再单独说了。

Singularity 容器默认就可以支持使用 GUI 程序，只需要在容器内安装好运行 GUI 程序所需的依赖即可，一般从命令行启动的时候会提示你缺少哪些库之类的，按图索骥，找到缺少的对应的包，全部装上去就能用了。不过好像用来运行一个完整的桌面不是很合适。

Podman 和 Docker 理论上是兼容的，不多说。

本文主要针对的是 Docker 容器。

本文也不对具体的方案做详细的介绍，只是指出大约有哪些方法可以考虑。

## 使用 x11docker

[x11docker](https://github.com/mviereck/x11docker) 可以帮助你运行 Docker 容器中的 GUI 程序，甚至是完整的桌面。x11docker 本身只是一个近万行的 bash 脚本，所以它的安装是非常简单的，只需要把 x11docker 文件下载，存放到 `~/bin` 目录下，并将 `~/bin` 加入你的 `PATH` 即可使用。

不过 x11docker 还是需要一些其他的依赖包，最小的依赖是你至少得在宿主机上安装 X server。不过官方建议你最好还要装上 `xpra` `Xephyr` `xinit` `xauth` `xclip` `xhost` `xrandr` `xdpyinfo` 这几个包。

更多详细的介绍请看官方文档。

## 使用 VNC

实际上，我们完全可以在 Docker 容器内安装完整的桌面环境，然后在里面启动 VNC 服务，最后我们通过 VNC 客户端访问到容器内的桌面环境。这种用法就比较类似于虚拟机的使用了，就好像我们有了一个完整的带桌面环境的 Linux 系统。

这种方案一般还会考虑使用 [noVNC](https://github.com/novnc/noVNC) 来提供网页访问，这样用户就连 VNC 客户端都不用了，直接通过浏览器访问即可。

在桌面环境的选择上，一般首推 Xfce4，其次是 LXDE，LXQt 等桌面，GNOME 和 KDE 一般不建议使用，因为不一定能用，而且即使能用也会很麻烦。

## 使用 xrdp

[xrdp](http://xrdp.org)  也是一种可选的远程桌面解决方案。与 Linux 上我们常用的 VNC 不同，xrdp 使用的是开源的 RDP 协议实现。RDP 协议实际上就是 Windows 的远程桌面所使用的协议。采用此方案的优点是对 Windows 用户更加友好，这回连客户端都不需要安装了，直接使用 Windows 远程桌面去连接即可。

但是个人体验发现实际的反应速度不如 VNC 流畅，不过要是在局域网内，这就不是问题了。

## 使用 xpra

[xpra](https://xpra.org) 可以看作是 GUI 版本的 screen，自然而然的也可以用在这个场景下。xpra 还有个优点，他支持多种类型的协议，甚至可以直接用 VNC 连接。同时还有一个不错的 HTML5 客户端，可以直接让用户通过网页来访问。

## 总结

目前来说，

* 对于宿主机有图形界面的情况，使用 x11docker 应该是比较简单的，需要的依赖包最少．
* 对于宿主机无图形界面或者宿主机图形界面访问不太方便的情况来说，使用 VNC 方法（特别是搭配 noVNC 实现通过网页访问）是比较普遍的选择．
* xrdp 的稳定性和速度还是比较一般的，主要还是受限于 RDP 协议的开源实现的性能吧．
* xpra 这个应该是一个比较小众的选择，即使不是在 Docker 环境，其他场景下也很少见到有人采用 xpra．

综上，个人建议宿主机有图形界面就用 x11docker，无图形界面就用 (no)VNC．