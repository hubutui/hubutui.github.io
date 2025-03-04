---
title: "在 Rocky Linux 上安装 Nvidia 显卡驱动和 CUDA 的最新最佳姿势"
date: 2025-03-04T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - Rocky Linux
  - Nvidia
  - V100
  - A100
  - CUDA
---

本文简单介绍在 Rocky Linux 9 上安装 Nvidia 显卡驱动的最新最佳方案．这个应该同样适用于兼容 RHEL 9 系列的发行版．

## 简介

简略介绍一下，Linux 下安装 Nvidia 显卡驱动主要有几种方式：

1. 使用 Nvidia 官方的 runfile 安装。基本上按照官方的操作来都会成功的，但是可能需要一些手动的配置，下载 runfile 可能也有点小问题。最后就是如果系统升级了（特别是内核升级了），又需要重新操作一遍。
2. 使用操作系统提供的驱动包，具体的又分为两种：
   1. 使用 dkms 即动态内核模块，缺点是安装的时候需要编译，而且可能编译没成功也没有任何提示。优点是一般都有默认的钩子来在系统更新内核的时候自动触发编译，一般不会因为升级内核导致驱动不可用。
   2. 不使用 dkms，也就是直接安装预编译好的内核包。缺点是如果打包者没有及时跟着内核版本的更新而重新打包更新驱动包，用户更新了内核就会出错。

个人建议使用 dkms 方案。这也是本文后续推荐使用的。

```bash
# 安装必备的开发工具，因为我们后面需要编译内核模块
dnf groupinstall -y "Development Tools"
dnf install -y pciutils elfutils-libelf-devel libglvnd-opengl libglvnd-glx libglvnd-devel acpid dkms
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
dnf install -y kernel-devel-matched kernel-headers
# 需要启用的 crb 源和 epel 源
dnf config-manager --set-enabled crb
dnf install -y epel-release
dnf module install -y nvidia-driver:latest-dkms/default
# 或者不用 dkms
# dnf module install -y nvidia-driver:latest/default
# 如果有问题需要重置，可以执行
dnf module reset nvidia-driver
# 可选，禁用开源驱动模块
grubby --args="nouveau.modeset=0 rd.driver.blacklist=nouveau" --update-kernel=ALL
# 如果你还需要安装 CUDA，这里请根据需要指定要安装的版本
dnf install cuda-12-8
```

Note：个人建议不用在全局安装 CUDA，而是使用在使用 conda 或者 Docker 的时候分别在 conda 环境或者 Docker 镜像内安装对应的 CUDA 包．

## 参考

1. Rocky Linux 的 Nvidia 显卡驱动[安装文档](https://docs.rockylinux.org/desktop/display/installing_nvidia_gpu_drivers)
2. Nvidia 官方的显卡驱动[安装文档](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html#network-repository-installation)
