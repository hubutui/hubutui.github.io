---
title: "使用 ArchLinux 进行深度炼丹"
date: 2018-10-22T00:00:00+08:00
---

## 缘起

见[此文](https://medium.com/@k_efth/deep-learning-in-arch-linux-from-start-to-finish-with-pytorch-tensorflow-nvidia-cuda-9a873c2252ed)有感而发．

目前大家深度炼丹普遍使用的是 Ubuntu 系统，看 CUDA 支持的发行版，自然也会有 Cent OS 和 openSUSE．实验室使用的也是 Ubuntu 系统，最终也才终于将全部机器都升级为 Ubuntu 16.04，用上了 systemd OS．

然而，Ubuntu 16.04 用起来还是各种麻烦，最难过的是软件包的版本不更新，遇到依赖问题比较难解决，安装 N 卡驱动也不够省心．

实际上用得最顺手的还是 Arch Linux．那么如何在 Arch Linux 上深度炼丹呢？So easy．只需要三步：安装 Arch Linux，安装 N 卡驱动和深度炼丹框架，开始炼丹．

## 安装 Arch Linux 和炼丹框架

安装 Arch Linux 这一步应该直接参考 Arch wiki 的安装指南，这里就不废话了．

### 关于 Arch Linux 的安装小声 bb

考虑到目前大多数主板都是用的 UEFI，建议分区使用 GPT 分区表，在固态硬盘上分一个 500M 左右的 FAT32 文件系统的 `boot` 分区，然后剩余空间丢给根分区，机械硬盘就全部分给 `home` 分区，如果后期有增加机械硬盘的打算，可以配 LVM，方便扩展．然后安装引导的时候，建议使用 [GRUB](https://wiki.archlinux.org/index.php/GRUB)．安装 GRUB 的命令为：

```shell
grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=GRUB
```

这里的 `esp` 应该就是 `/boot` 分区．

### 安装驱动和其他软件

安装 N 卡驱动和炼丹框架：

```shell
pacman -S nvidia
pacman -S cuda
pacman -S tensorflow-opt-cuda
pacman -S python-pytorch-cuda
pacman -S pycharm-community-edition
```

简单的几条命令，直接就安装好最新版本的驱动和 cuda 以及 tensorflow 和 pytorch 炼丹工具了．

## 进化

什么？你从 github 上搬运的代码对炼丹工具有版本要求？那就只好上 Anaconda 了呀．先添加 [Arch Linux CN 源](https://www.archlinuxcn.org/archlinux-cn-repo-and-mirror/)．然后安装 Anaconda，即可根据需要使用 conda 创建虚拟环境，把所有的 python 包都装到虚拟环境中．而且，根据 Anaconda 的官方[博文](https://www.anaconda.com/blog/developer-blog/tensorflow-in-anaconda/)，Anaconda 会自动根据炼丹框架的需要装好所需的 cuda 版本，从此不再有 cuda 版本错误或者不匹配的烦恼．

Arch Linux CN 源提供的 anaconda 包默认安装到 `/opt/anaconda` 目录下，激活基本环境请直接使用 `source /opt/anaconda/bin/activate`，然后就可以使用 conda 创建和管理你的虚拟环境，开始愉快地炼丹咯．

### 其他需要注意的地方

众所周知，Arch Linux 的包都是最新版本的，编译器也是．所以，如果你的代码需要使用特定版本的编译器，可以看看 AUR 中提供的那些，gcc 版本从 4.9 到 6.4 都可以用，然后在编译的时候注意指定编译器即可．

## 有始有终

更多信息，请多多参考 Arch Linux 的官方 wiki．
