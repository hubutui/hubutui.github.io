---
title: "如何在非 Archlinux 上安装 pacman 然后用于给 ArchLinux 打包软件"
date: 2025-11-20T15:56:01+08:00
draft: false
toc: false
images:
tags:
  - archlinux
  - pacman
---

## 前因

手上有几台 CPU 核心较多的服务器暂时闲置，考虑用于编译打包 ArchLinux 的软件包，相比本地电脑具有更高的性能且不会占用本地资源。然而服务器运行的是 Rocky Linux 9.5 而非 ArchLinux，无法直接用于打包工作。

一般我们使用 [devtools](https://archlinux.org/packages/extra/any/devtools) 中的构建脚本在 chroot 环境中进行构建。这实际上是通过 systemd-container 在容器实现的。如果想要在 Docker 容器里再套一层 systemd-container，又会增加复杂性，而且也还需是需要切换到非 root 用户操作的。

实际上，Rocky Linux 支持安装 pacman。只要安装了 pacman，就可以使用它来安装其他构建工具，从而实现在非 ArchLinux 系统上打包软件的目标。

## 初步想法

这里不提供过于详细的步骤，而是给出一个整体方案。

要为 ArchLinux 打包软件，通常需要使用 devtools 提供的命令，如 `extra-x86_64-build`。我个人习惯使用其 fork 版本 `devtools-archlinuxcn`。通过查询可以得知其依赖关系：

```text
软件库         : archlinuxcn
名字           : devtools-archlinuxcn
版本           : 1:1.3.0-2
描述           : Tools for Arch Linux package maintainers, archlinuxcn fork
架构           : any
URL            : https://gitlab.archlinux.org/archlinux/devtools
软件许可       : GPL-3.0-or-later
组             : 无
提供           : devtools=1.3.0-2
依赖于         : arch-install-scripts  awk  bash  binutils  coreutils  curl  diffutils  expac  fakeroot  findutils  glow  grep  gum  jq
                 openssh  parallel  rsync  sed  util-linux  breezy  git  mercurial  subversion
可选依赖       : btrfs-progs: btrfs support
                 bat: pretty printing for pkgctl search
                 nvchecker: pkgctl version subcommand
与它冲突       : devtools
取代           : 无
下载大小       : 127.48 KiB
安装后大小     : 437.50 KiB
打包者         : lilac (on behalf of f2q9sf79f9owdg2o) <f2q9sf79f9owdg2o@chyen.cc>
编译日期       : 2024年12月18日 星期三 20时17分43秒
验证者         : SHA-256 校验值
```

其中只有 `arch-install-scripts` 是 ArchLinux 特有的软件包：

```text
软件库         : extra
名字           : arch-install-scripts
版本           : 31-1
描述           : Scripts to aid in installing Arch Linux
架构           : any
URL            : https://gitlab.archlinux.org/archlinux/arch-install-scripts
软件许可       : GPL-2.0-only
组             : 无
提供           : 无
依赖于         : awk  bash  coreutils  grep  pacman  util-linux
可选依赖       : 无
与它冲突       : 无
取代           : 无
下载大小       : 16.59 KiB
安装后大小     : 39.16 KiB
打包者         : Morten Linderud <foxboron@archlinux.org>
编译日期       : 2025年10月01日 星期三 05时17分04秒
验证者         : SHA-256 校验值  数字签名
```

此外，还可能需要 `pacman-contrib` 包中的 `updpkgsums` 命令，用于自动更新 PKGBUILD 文件中的校验和：

```text
软件库         : extra
名字           : pacman-contrib
版本           : 1.13.0-1
描述           : Contributed scripts and tools for pacman systems
架构           : x86_64
URL            : https://gitlab.archlinux.org/pacman/pacman-contrib
软件许可       : GPL-2.0-or-later
组             : 无
提供           : 无
依赖于         : pacman
可选依赖       : diffutils: for pacdiff
                 fakeroot: for checkupdates
                 findutils: for pacdiff --find
                 mlocate: for pacdiff --locate
                 plocate: faster mlocate alternative
                 perl: for pacsearch
                 sudo: privilege elevation for several scripts
                 vim: default diff program for pacdiff
                 neovim: default diff program for pacdiff if EDITOR=nvim
与它冲突       : 无
取代           : 无
下载大小       : 49.43 KiB
安装后大小     : 129.30 KiB
打包者         : Daniel M. Capella <polyzen@archlinux.org>
编译日期       : 2025年08月27日 星期三 07时51分42秒
验证者         : SHA-256 校验值  数字签名
```

其中 pacman 需要使用 Rocky Linux 的包管理工具 dnf 安装，其他软件包则可以通过 pacman 安装。

因此，整体方案为：

1. 使用 dnf 安装 pacman
2. 使用 pacman 更新自身到较新版本
3. 使用更新后的 pacman 安装 arch-install-scripts、pacman-contrib、devtools 等软件包

**注意**：使用 pacman 安装软件包时应添加 `-Sdd` 参数忽略依赖检查，避免安装 Rocky Linux 本身已提供的依赖包，以免造成系统混乱。

## 详细步骤

### 安装 pacman

首先使用 dnf 安装 pacman：

```bash
dnf install -y pacman
```

安装过程日志示例（已精简）：

```text
Last metadata expiration check: 3:19:55 ago on Thu 20 Nov 2025 01:24:13 PM CST.
Dependencies resolved.
=====================================================================================================
 Package                             Architecture  Version                    Repository        Size
=====================================================================================================
Installing:
 pacman                              x86_64        6.0.2-3.el9                epel             724 k
Installing dependencies:
 archlinux-keyring                   noarch        20240709-1.el9             epel             1.2 M
 bsdtar                              x86_64        3.5.3-6.el9_6              appstream         62 k
 keyrings-filesystem                 noarch        1-15.el9                   epel             7.5 k
 libalpm                             x86_64        6.0.2-3.el9                epel             257 k
 pacman-filesystem                   noarch        6.0.2-3.el9                epel             7.6 k
 ...
Installing weak dependencies:
 arch-install-scripts                noarch        28-2.el9                   epel              29 k
```

可以看到系统会自动安装 `archlinux-keyring`（包含用于验证软件包签名的 PGP 密钥）和 `arch-install-scripts`。

### 配置 pacman

由于 dnf 安装的 pacman 版本较旧，配置文件中仍包含已废弃的 community 仓库（该仓库已合并至 extra）。需要手动编辑 `/etc/pacman.conf`，删除 community 仓库相关配置。

同时建议修改 `/etc/pacman.d/mirrorlist`，更换为国内镜像源以提升下载速度。

**注意**：不建议直接更新 pacman 到最新版本。若新版本依赖的库版本高于 Rocky Linux 提供的版本，会导致 pacman 无法正常使用。实际上我们只需要 devtools 在容器中使用新版软件包，pacman 本身使用旧版本即可满足需求。

### 添加 archlinuxcn 仓库

按照 [ArchLinuxCN](https://github.com/archlinuxcn/repo) 官方文档配置：

1. 编辑 `/etc/pacman.conf`，添加以下内容：

```text
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```

2. 安装 archlinuxcn 密钥环：

```bash
pacman -Syy
pacman -S --overwrite='*' archlinuxcn-keyring archlinux-keyring
```

### 安装必要的依赖

使用 dnf 安装构建过程中需要的工具：

```bash
dnf install -y fakeroot systemd-container
```

### 安装构建工具

使用 pacman 安装 pacman-contrib 和 devtools：

```bash
pacman -Sdd pacman-contrib devtools-archlinuxcn devtools-cn-git
```

### 配置 sudo 权限

由于构建过程不允许以 root 身份执行，需要配置 sudo，允许普通用户执行构建命令。示例配置如下：

```bash
visudo
```

在打开的配置文件中添加（假设用户名为 tom）：

```text
tom ALL=(ALL) NOPASSWD:ALL
```

### 测试构建

现在可以使用普通用户开始打包测试。以 flameshot 为例：

```bash
# 克隆软件包仓库
pkgctl repo clone --protocol=https flameshot
cd flameshot

# 使用 extra-x86_64-build 构建
extra-x86_64-build

# 或使用 archlinuxcn-x86_64-build 构建（会从 archlinuxcn 源安装依赖）
archlinuxcn-x86_64-build
```

**说明**：这里使用 `extra-x86_64-build` 而不是 `pkgctl build`，因为旧版本 pacman 不兼容 `pkgctl build` 新增的某些选项。

## 进阶改进

前面提到不建议更新 pacman 到最新版本，这是因为 pacman 默认采用动态链接方式编译，其运行时会依赖宿主系统提供的共享库。当 Rocky Linux 提供的库版本低于 ArchLinux 打包的 pacman 所需版本时，就会出现兼容性问题导致 pacman 无法正常运行。

EPEL 仓库虽然提供了 pacman，但版本较旧（6.0.2），缺少很多新特性。

为了在不破坏系统稳定性的前提下使用最新版 pacman，我们可以采用**静态链接**的方案。静态链接会将所有依赖库的代码直接编译到可执行文件中，生成的二进制文件不再依赖宿主系统的共享库，从而完全避免了库版本冲突问题。

[pacman-static](https://aur.archlinux.org/packages/pacman-static) 就是一个静态链接版本的 pacman。不过，原版 pacman-static 的 PKGBUILD 为了与系统的 pacman 共存做了一些修改（移除配置文件、重命名二进制文件添加 `-static` 后缀等）。由于我们的目标是用它完全替代 EPEL 的 pacman，需要恢复这些内容。

### 构建 pacman-static

首先在一台 ArchLinux 机器上克隆 pacman-static 的 PKGBUILD：

```bash
git clone https://aur.archlinux.org/pacman-static.git
```

修改 PKGBUILD 文件：

1. 从 `depends` 数组中移除 `pacman`（避免依赖冲突）
2. 修改 `package()` 函数，保留配置文件和原始的二进制文件名：

```bash
package() {
    cd "${srcdir}"/pacman
    DESTDIR="${pkgdir}" meson install -C build
    # 这里移除了 pacman 的配置文件，重命名了 pacman 的可执行文件
    # 我们注释掉，保留原样
    # rm -rf "${pkgdir}"/usr/share "${pkgdir}"/etc
    # for exe in "${pkgdir}"/usr/bin/*; do
    #     if [[ -f ${exe} && $(head -c4 "${exe}") = $'\x7fELF' ]]; then
    #         mv "${exe}" "${exe}"-static
    #     else
    #         rm "${exe}"
    #     fi
    # done

    cp -a "${srcdir}"/temp/usr/{bin,include,lib} "${pkgdir}"/usr/lib/pacman/
    sed -i "s@${srcdir}/temp/usr@/usr/lib/pacman@g" \
        "${pkgdir}"/usr/lib/pacman/lib/pkgconfig/*.pc \
        "${pkgdir}"/usr/lib/pacman/bin/*
}
```

保存修改后，开始构建：

```bash
cd pacman-static
# 使用 --skippgpcheck 跳过 PGP 签名验证
extra-x86_64-build -- -- --skippgpcheck
```

构建完成后会生成类似 `pacman-static-7.0.0.r6.gc685ae6-19-x86_64.pkg.tar.xz` 的软件包文件。

### 安装到 Rocky Linux

将构建好的软件包复制到 Rocky Linux 机器上，然后安装：

```bash
pacman -U --overwrite='*' pacman-static-7.0.0.r6.gc685ae6-19-x86_64.pkg.tar.xz
```

**配置 SSL 证书路径**

由于 ArchLinux 和 Rocky Linux 使用不同的 SSL 证书文件名，需要创建一个软链接：

```bash
ln -s /etc/ssl/certs/ca-bundle.crt /etc/ssl/certs/ca-certificates.crt
```

**验证安装**

检查 pacman 版本，确认已更新到最新版：

```bash
pacman --version
```

最后重新编辑 `/etc/pacman.conf`，启用需要的仓库。现在就可以使用最新版 pacman 了。

## 总结

本文介绍了在非 ArchLinux 系统（以 Rocky Linux 为例）上安装 pacman 并配置 ArchLinux 打包环境的方法。相比 Docker 容器方案，这种方式更加轻量，避免了容器嵌套的复杂性。

### 核心方案

**基础方案**：

1. 利用 EPEL 仓库提供的 pacman 包作为基础
2. 通过 pacman 安装 ArchLinux 特有的构建工具
3. 使用 `-Sdd` 参数避免依赖冲突

基础方案使用 EPEL 提供的旧版本 pacman（6.0.2），虽然版本较旧，但对于大多数打包需求已经足够，且与宿主系统兼容性最好。

**进阶方案**：

对于需要使用最新 pacman 的场景，可以通过修改打包 pacman-static 来获取最新版本。pacman-static 采用静态链接方式，将所有依赖库打包到二进制文件中，从而避免了动态链接版本的库依赖问题。

### 关键注意事项

- 使用 `pacman -Sdd` 忽略依赖检查，避免安装与宿主系统冲突的库
- 基础方案保持使用旧版 pacman，确保稳定性；进阶方案使用静态链接的最新版
- 构建命令需要使用普通用户权限，需正确配置 sudo
- 进阶方案需要配置 SSL 证书软链接以确保 HTTPS 连接正常
- `pkgctl --build` 实测还是无法使用，但是一般使用 `extra-x86_64-build` 或者 `archlinuxcn-x86_64-build` 也足够满足打包需求

### 适用范围

通过这种方式，可以充分利用闲置服务器资源进行软件包编译，无需在本地占用资源。

此外，对于 Debian/Ubuntu 系统，也可以采用类似的方法，只是对应的 pacman 包名为 `pacman-package-manager`，其他操作步骤基本一致。
