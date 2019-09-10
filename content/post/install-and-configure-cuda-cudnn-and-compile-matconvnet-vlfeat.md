---
title: "Linux 环境下安装 CUDA & CuDNN，并编译安装 MatConvNet & VLFeat"
date: 2017-10-26T00:00:00+08:00
---

## 前言

本文针对是 Arch Linux & Ubuntu，其他发行版也可以参考．

## 版本信息

* Ubuntu 16.04.3, gcc 5.4.0, CUDA 8.0, CuDNN 5.1, matconvnet 1.0-beta24, vlfeat 0.9.20

## 安装 CUDA 和 CuDNN
### Arch Linux

Arch Linux 的 community 源里就已经有打包好的，目前的版本为 cuda 9.0.176-4, cudnn 7.0.3-1，直接安装即可：

```shell
pacman -S cuda cudnn
```

这个安装包比较大，如果下载比较慢的，可以用以下命令输出安装包的 url 地址：

```shell
pacman -Sp cuda cudnn
```

根据 url 地址，用其他方法下载到本地之后，再用 `pacman -U pkgname.pkg.tar.xz` 命令安装．

### Ubuntu

Ubuntu 可以直接从 Nvidia [官网](https://developer.nvidia.com/cuda-downloads)下载．推荐下载离线包，例如对应 Ubuntu 16.04 的 [cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64.deb](https://developer.nvidia.com/cuda-downloads)．下载完成后使用 `dpkg` 命令安装：

```shell
sudo dpkg -i cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64.deb
```

按照提示添加 apt key．然后使用 apt 命令安装：

```shell
apt install cuda
```

注意，如果之前安装过 Ubuntu 源里的低版本 CUDA 的话，要先卸载掉．

然后是安装 CuDNN，需要在 Nvidia 官网注册帐号才能下载 CuDNN．对 Ubuntu 来说，只需要下载 libcudnn, libcudnn-doc, libcudnn-dev, libcudnn-doc 三个包，然后用 `dpkg` 命令安装即可．其他的发行版若没有打包，Nvidia 也没有打包的，也可以下载 cudnn tarball for Linux，然后解压，复制文件到相应的目录即可：

```shell
tar xfv cudnn-8.0-linux-x64-v5.1.tgz
sudo cp -P cuda/include/cudnn.h /usr/local/cuda/include
sudo cp -P cuda/lib64/libcudnn* /usr/local/cuda/lib64
sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*
```

注意到上面的 `cp` 命令添加了 `-P` 选项，表示不跟随软链接．值得一提的是，deb 包默认会将 cudnn 安装到 `/usr` 目录下，个人比较推荐使用 tarball 安装，即解压后复制粘贴相应的文件到合适的目录下．

在以上步骤中若有提示依赖缺失，软件包尚未配置好的，还需要执行命令：

```shell
sudo apt install -f
```

自动修复相关的依赖，然后应该就没有问题了．

### 多版本 CUDA 和 CuDNN 共存

Ubuntu 的 CUDA 默认安装位置为 `/usr/local/cuda-<version>`，这样子的话，实际上是可以安装多个版本的 CUDA 的．至于 CuDNN 的多版本共存，则需要直接下载 Linux 版本的包，而不能直接使用 deb 包安装了．解压下载好的 tarball，复制到合适的目录下即可．在编译 MatConvNet 的时候根据需要设置变量 `CUDNN_ROOT`．

### 环境变量设置

安装完 CUDA 和 CuDNN 之后，你可能想要设置以下环境变量，将它们添加到 `~/.bashrc` (bash) 或者 `~/.zshrc` (zsh) 中：

```shell
export CUDA_HOME=/usr/local/cuda-8.0
export CUDAHOME=$CUDA_HOME
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-8.0/cudnn-5/lib64:/usr/local/cuda-8.0/cudnn-6/lib64
```

## 编译 MatConvNet

从[官网](http://www.vlfeat.org/matconvnet/)下载合适版本的 MatConvNet 源码包，目前版本为 1.0-beta25．解压后进入源码目录：

```shell
tar xfv matconvnet-1.0-beta24.tar.gz
cd matconvnet-1.0-beta24
```

然后直接编译：

```shell
# Arch Linux
make ENABLE_GPU=yes ENABLE_CUDNN=yes CUDAROOT=/opt/cuda CUDNNROOT=/opt/cuda MATLABROOT=/usr/local/MATLAB/R2017a ARCH=glnxa64

# Ubuntu
make ENABLE_GPU=yes  CUDAROOT=/usr/local/cuda-8.0 MATLABROOT=/usr/local/MATLAB/R2017a ENABLE_CUDNN=yes CUDNNROOT=/usr/local/cuda-8.0 ARCH=glnxa64 -j 8
```

这里直接启用了 CUDA 和 CuDNN 支持，不同发行版的目录可能不一样，按需修改变量 `CUDAROOT` 和 `MATLABROOT` 即可．

## 测试

编译完成之后可以在 MATLAB 中测试一下是否成功编译，输入命令 `vl_testnn`；或者输入 `vl_testnn('gpu', true)` 测试 GPU 的支持．

## 编译安装 VLFeat

这个其实也很简单，直接从[官网](http://www.vlfeat.org)下载最新的 VLFeat 源码包，目前为 0.9.20 版本．接着就是几个命令：

```shell
tar xfv vlfeat-0.9.20.tar.gz
cd vlfeat-0.9.20
make ARCH=glnxa64 MEX=/usr/local/MATLAB/R2017a/bin/mex
```

mex 命令的路径可以根据需要修改．

## 参考

* MatConvNet 官方[安装教程](http://www.vlfeat.org/matconvnet/install/)
* MatConvNet 官方的[命令行编译教程](http://www.vlfeat.org/matconvnet/install-alt/)
* VLFeat 官方[安装教程](http://www.vlfeat.org/compiling-unix.html)
* CUDA 官方[安装教程](http://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html)
* CuDNN 官方[安装教程](http://docs.nvidia.com/deeplearning/sdk/cudnn-install/index.html#installlinux-tar)
