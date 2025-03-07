---
title: "netboot.xyz 简明教程"
date: 2025-03-07T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - netboot.xyz
  - netboot
  - PXE
  - iPXE
---

## 前言

最近服务器出现故障需要维修，每次维修要么将本地下载好的一个 1G 左右大小的 ISO 文件通过带外管理工具挂载上去，要么做一个 U 盘启动盘去机房插到机器上．前者受限于本地机器与带外管理工具的传输速度，后者跑机房总归有些麻烦．使用带外管理工具本来就是为了免去奔波机房的麻烦，但是随之而来的网络传输速度的限制，导致其实要使用 LiveCD 维护系统的时候速度慢得让人难以忍受．PS：这个传输速度可能并不是说你正在使用的电脑与带外管理工具之间的传输速度的问题，也可能是带外管理工具本身就比较糟糕的 IO．

天佑我等，大佬给我们安利了 `netboot.xyz`．这是一个让你可以从网络启动系统安装器或者其他工具的小巧便捷的工具．例如前面我们遇到的这种情况，我们只需要从 [netboot.xyz](https://netboot.xyz) 下载好一个不到 2.5M 的文件 [netboot.xyz.iso](https://boot.netboot.xyz/ipxe/netboot.xyz.iso)，通过带外管理工具挂载，然后启动的时候选择这个镜像启动，即可进入 netboot.xyz 的启动菜单．在 netboot.xyz 的菜单上，你可以根据选择启动不同的操作系统，netboot 将会从网络下载启动这个系统所需的最小的文件，载入内存，然后直接启动．

![image](https://netboot.xyz/assets/images/netboot.xyz-d976acd5e46c61339230d38e767fbdc2.gif)

## 详细一些的介绍

### Linux 网络安装器菜单

一些 Linux 操作系统提供了可以通过网络启动的安装器，这个网络安装启动器其实是一种比较轻量级的安装方式，因为他只需要检索和获取一组最小的安装程序内核，然后根据需要安装软件包．netboot.xyz 把这些安装器都集成在了 `Distributions / Linux Network Installs` 菜单下．支持使用这种方式启动的安装器的这些常见的 Linux 发行版：

1. ArchLinux
2. CentOS Stream
3. Debian
4. openSUSE
5. Rocky Linux
6. Red Hat Enterprise Linux
7. Rocky Linux
8. Ubuntu

除此之外还有别的 Linux 发行版也是支持的，但是一般人用的很少超出以上的列表了．

进入菜单之后，我们直接选择想要的系统即可进去开始安装了．这些菜单默认都是从各个 Linux 发行版的官网下载安装器的，虽然这个文件相对于完整的 ISO 来说是小了很多，但是如果从访问官网的速度比较慢，还是要设置使用国内的镜像源会更快一些．

这个时候，我们可以先在 iPXE shell 里设置环境变量来指定相应的环境变量．但是具体每个 Linux 发行版使用的环境变量名字是不同的，一般我们需要关注的是 `xxx_mirror` 和 `xxx_base_dir` 这两个环境变量．具体的，我们可以查阅 netboot.xyz 的 github 仓库里的脚本，这些脚本都在这个[目录](https://github.com/netbootxyz/netboot.xyz/tree/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu)下．例如，对于 ArchLinux，对应的脚本为 [archlinux.ipxe.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/archlinux.ipxe.j2)．我们需要检查和设置 `archlinux_mirror` 和 `archlinux_base_dir` 这两个环境变量．

进入 iPXE shell 之后：

```bash
# 先获取默认的值看看是什么
show archlinux_mirror
# 输出为 archlinuxz_mirror:string = mirrors.kernel.org
show archlinux_base_dir
# 输出为 archlinuxz_base_dir:string = archlinux
```

注意看，这里的 `archlinux_mirror` 的值是不带 `http(s)://` 前缀的，有的发行版是带的．我们可以设置为使用中国科技大学的开源镜像．只需要：

```bash
set archlinux_mirror mirrors.ustc.edu.cn
```

特别注意，默认的 netboot.xyz 带的 iPXE 是不带 HTTPS 支持的．因此，如果你使用的开源镜像源不支持 HTTP 访问则无法使用．比如清华大学的开源镜像源在这里就不能用，但是中科大的是 OK 的．设置完毕之后输入 `exit` 命令退出 iPXE SHELL，然后重新在 Linux Network Installs 菜单下选择 ArchLinux 启动，这次应该会很快下载好所需的文件并启动了．

又如 Rocky Linux，他的脚本在 [rockylinux.ipex.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/rockylinux.ipxe.j2)．根据这个脚本，我们可以知道我们要设置的环境变量为 `rockylinux_mirror` 和 `rockylinux_base_dir`：

```bash
show rockylinux_mirror
# 输出为 rockylinux_mirror:string = http://download.rockylinux.org
show rockylinux_base_dir
# 输出为 rockylinux_base_dir:string = pub/rocky
```

这次我们试着用南方科技大学的开源镜像

```bash
set rockylinux_mirror http://mirrors.sustech.edu.cn
set rockylinux_base_dir rocky-linux
```

注意：

1. 这里的 `rockylinux_mirror` 要带 `http://` 前缀的，否则会下载失败．
2. 这里的 `rockylinux_base_dir` 要设置为 `rocky-linux`，因为南方科技大学开源镜像是把 Rocky Linux 的内容放在了 `rocky-linux` 路径下．

再如 Ubuntu，他的脚本在 [ubuntu.ipxe.j2](https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/ubuntu.ipxe.j2)．易知他需要设置的环境变量为 `ubuntu_mirror` 和 `ubuntu_base_dir`．具体的设置与前文的类似，这里不再赘述了．

如果 `xxx_base_dir` 设置出错，下载的是会输出他实际下载用的 URL，根据这个信息去开源镜像站点检查，然后重新设置为正确的值即可．注意每次进入 iPXE shell 都要同时设置这两个环境变量．

#### 更简洁一些的方法

实际上，这些 iPXE 脚本我们可以自己维护，然后在 iPXE shell 中直接用 `chain` 命令来启动即可．例如，我们把 `archlinux.ipxe.j2` 脚本下载保存在本地，并且编辑其中的内容，提前设置好 `archlinux_base_dir` 和 `archlinux_mirror` 环境变量，然后在 iPXE Shell 中使用 chain 命令启动：

```bash
chain url
```

这里的 `url` 为 `archlinux.ipxe.j2` 的 URL．我们可以在本地使用 darkhttpd 或者其他类似的工具轻松地启动一个 HTTP 服务器，拿到这个 URL 输入到上面的命令行即可．

你可以自己维护自己常用的 Linux 发行版的对应的脚本，然后就可以直接通过简单的 `chain` 命令启动了．

对于大多数的维护工作，本文读到这里已经足够了．后面的部分并不是必需的．

### Linux LiveCD 菜单

很多 Linux 发行版还会提供一个很大的 ISO 文件或者 Live CD/DVD，让你可以直接用来安装或者体验系统．但是这些文件太大了，不适合网络启动．对于这种情况，netboot.xyz 会定期检查这些系统的更新，然后重新打包发布到 Github 上，然后让 netboot.xyz 去下载和载入这些重新打包过的文件来通过网络启动．

这些内容都放在了 LiveCDs 菜单下，这些发行版包括我们常用的 Debian, Fedora，Ubuntu 等．如果你想要用这些，就需要保证你的网络能够正常访问 Github．他们也有对应的 iPXE 脚本，放在这个[目录](https://github.com/netbootxyz/netboot.xyz/tree/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu)下，都是以 `live` 开头的那些．

例如，根据 [https://github.com/netbootxyz/netboot.xyz/blob/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu/live-ubuntu.ipxe.j2](live-ubuntu.ipxe.j2)，我们可以知道他需要从 `live_endpoint` 下载文件，默认的 `live_endpoint` 为 `https://github.com/netbootxyz`．把脚本中的 `kernel_url` 手动选一下，我们可以找到 [netbootxyz/ubuntu](https://github.com/netbootxyz/ubuntu-squash) 这个仓库，打开其 Release 页面，可以看到实际需要下载 `vmlinuz`，`initrd` 等文件．如果我们提前下载好，并且设置按照相同的目录结构准备好，启动一个 HTTP 服务，进入 iPXE shell 设置一下 `live_point` 为我们本地的地址，那么实际下载也就从本地的服务器下载，应该会比从 Github 上下载要快很多了．

事实上，我们可以用官方提供的 docker compose 例子来使用．我们可以使用 [docker-netbootxyz](https://github.com/netbootxyz/docker-netbootxyz)．具体来说：

```bash
git clone git@github.com:netbootxyz/docker-netbootxyz.git
cd docker-netbootxyz
cp docker-compose.yml.example docker-compose.yml
```

编辑修改 `docker-compose.yml` 中的 `volumes` 设置，即数据存放的位置．例如：

```yaml
---
services:
  netbootxyz:
    image: ghcr.io/netbootxyz/netbootxyz
    container_name: netbootxyz
    environment:
      - MENU_VERSION=2.0.84 # optional
      - NGINX_PORT=80 # optional
      - WEB_APP_PORT=3000 # optional
    volumes:
      - ./volumes/config:/config # optional
      - ./volumes/assets:/assets # optional
    ports:
      - 3000:3000 # optional, destination should match ${WEB_APP_PORT} variable above.
      - 69:69/udp
      - 8080:80 # optional, destination should match ${NGINX_PORT} variable above.
    restart: unless-stopped
```

然后使用 `docker compose up -d` 命令启动，启动成功之后浏览器访问 `http://IP:3000` 即可看到网页形式的配置页面，点击切换到 `Local Assets`，左侧显示的是远程的文件，找到我们需要用的 Linux 发行版，勾选拉取，即可在右侧显示出来．浏览器访问 `http://IP:8080` 你应该就可以看到这些内容了．这个时候，只需要把 `live_endpoint` 设置为 `http://IP:8000`，那 netboot.xyz 就是从我们本地的这个服务器下载对应的文件了．

## 自托管的 netboot.xyz

实际上，前面我们已经稍微提到了一点，我们可以自己托管一个 netboot.xyz 的服务，这样我们启动进入 netboot.xyz 之后就不需要从互联网上下载内容，而是直接从我们自托管在局域网里的站点下载，速度肯定会更加快．

具体的文档可以参考[这里](https://netboot.xyz/docs/selfhosting)，我个人暂时没有自托管的需求，就没有深入研究．其实能够自托管会更加好，因为实际上进入 netboot.xyz 之后，菜单项也都是从 `https://boot.netboot.xyz/menu.ipxe` 获取的，如果无法访问此网站，那就不太好操作了．好在目前 GFW 还没有禁止访问此网站．

此外，如果你的系统有配置好的 iPXE，其实也不用下载 netboot.xyz 的 ISO 挂载到系统里然后启动，完全可以直接在 iPXE shell 里使用 `chain` 命令启动的．更多的启动 netboot.xyz 的方法可以参考[官方文档](https://netboot.xyz/docs/category/booting-methods)的介绍．

最后，补充一下网络设置．当我们启动 netboot.xyz 的时候，他默认会尝试可用的网络设备，并使用 DHCP 方式配置网络，如果配置失败，则需要用户手动配置，或者直接在启动的时候按 `m` 键进入 Failsafe menu 之后选择 Manual network configuration 进行手动配置网络．比如你的设备需要使用静态 IP 配置，只需要按照提示设置 IP、子网掩码、网关、DNS 服务器即可，这些信息应该由网络管理员提供．

## 总结

1. 对于大多数情况，我们需要的就是默认的 netboot.xyz 的 ISO 文件：[netboot.xyz.iso](https://boot.netboot.xyz/ipxe/netboot.xyz.iso) 来启动 netboot.xyz．
2. 在进入 netboot.xyz 之后，选择 iPXE shell 来设置环境变量 `xxx_mirror` 和 `xxx_base_dir`，其中 `xxx` 为你使用的发行版，请根据 netboot.xyz 的 iPXE 脚本里的名字来写．设置完毕之后退出 iPXE shell．
3. 选择进入 Linux Network Installs，选择你要使用的 Linux 发行版即可启动对应的安装环境．
4. 也可以直接预先编辑和托管 [netboot.xyz/menu](https://github.com/netbootxyz/netboot.xyz/tree/ada528e3ea16206321e352bb639816b9a6de8e31/roles/netbootxyz/templates/menu) 的这些 iPXE 脚本，提前设置好其中的 `xxx_mirror` 和 `xxx_base_dir`．在 iPXE shell（不论是 netboot.xyz 里的还是你的服务自己提供的）里使用 `chain` 命令启动即可．
5. 不太推荐使用 netboot.xyz 里提供的 Live CDs 来安装，因为要从 Github 下载文件，可能网络受限，不会很顺利．
