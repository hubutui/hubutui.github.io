---
title: "如何使用在 Anaconda 中使用 nvcc 编译源码——以编译 apex 为例"
date: 2019-09-14T00:00:00+08:00
draft: false
---

## 简介

apex 是 NVIDIA 开发的 PyTorch 扩展工具，支持混合精度训练、分布式训练以及 Sync BN 等．它需要从源码编译安装，正好用来在 LXD 容器内测试一下．我们的 LXD 容器内提供了 Miniconda，而 Miniconda 不含 nvcc．为此，我们从宿主机挂载了安装好的 CUDA 到 `/usr/local` 目录下供用户使用其中的 nvcc 编译器．

TIPS：本文对于未有使用 LXD 容器的用户同样适用．

## 安装步骤

### 使用 CUDA 10.0

我们先检查 LXD 容器内的 CUDA 和 CUDNN 的版本：

```shell
ls -lh /usr/local/cuda-10.0/lib64/libcudart.*
ls -lh /usr/local/cuda-10.0/lib64/libcudnn.*
```

查得具体的版本是 CUDA 10.0.130 和 CUDNN 7.3.1．

首先，我们创建一个 conda 虚拟环境，并直接指定需要安装的各个包及其版本：

```shell
conda create -n apex-cuda10 python=3.7 pytorch torchvision cudatoolkit=10.0 cudnn=7.3.1
```

大部分教程或者代码会默认用户将 CUDA 安装到 `/usr/local/cuda`．而我们为了安装多版本的 CUDA，实际的安装位置为 `/usr/local/cuda-10.0`，因此我们需要创建一个软链接（可以理解为快捷方式）指向我们所需的版本．此外，部分程序或者代码会检查环境变量 `CUDA_HOME`，并认为 CUDA 安装在该目录下．因此也可以设置该环境变量．

```shell
# 创建软链接
ln -s /usr/local/cuda /usr/local/cuda-10.0
```

这个命令需要管理员权限，请自己在命令之前加上 `sudo`．

**TIPS：**负责任的人不会直接给你一个带有 `sudo` 的命令，也不会建议你直接使用 `root` 用户操作．网络所有让你直接用 `root` 用户或者直接贴给你一个带 `sudo` 的命令的都是垃圾，绝对要先检查清楚．不知道该命令有什么作用最好就不要执行．

接着，从[官方主页](https://github.com/NVIDIA/apex)下载源码，这里直接用 `git clone`：

```shell
git clone https://github.com/NVIDIA/apex
cd apex
# 激活配置 conda 虚拟环境，在虚拟环境内安装
conda activate apex-copy
pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" .
```

如果你不想创建指向 `/usr/local/cuda-10.0` 的软链接，也那就设置环境变量 `CUDA_HOME`，PyTorch 的扩展代码一般默认会读取这个环境变量，Keras、TensorFlow 和 MXNet 等其他框架请参考其文档．

```shell
CUDA_HOME=/usr/local/cuda-10.0 -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" .
```

### 使用 CUDA 9.0

如果你想使用 CUDA 9.0，那么需要修改默认的编译器版本为 gcc/g++ 5．具体的 CUDA 对应的 gcc 版本对应关系请查阅文档．

```shell
# 我们的文档有提到过，LXD 容器内提供了多版本的 gcc/g++，使用一下命令按照提示修改即可
update-alternatives --config gcc
```

同样地，我们用前述方法查出版本为 CUDA 9.0.176 和 CUDNN 7.3.1．接着，我们创建一个虚拟环境并安装所需的软件包：

```shell
conda create -n apex-cuda9 python=3.7 pytorch torchvision cudatoolkit=9.0 cudnn=7.3.1
```

然后同样的方法进行编译和安装 apex：

```shell
git clone https://github.com/NVIDIA/apex
cd apex
# 激活配置 conda 虚拟环境，在虚拟环境内安装
conda activate apex-cuda9
CUDA_HOME=/usr/loca/cuda-9.0 pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" .
```

## 测试

安装完毕之后跑一下 apex 的例子测试一下．

```shell
cd examples/dcgan
# 测试一下 FP32 位精度的训练
python main_amp.py --opt_level O0 --niter 1
# 测试一下混合精度训练
python main_amp.py --opt_level O1 --niter 1
# 另外一种混合精度训练，跟 O1 相比更多的操作用 FP16 实现，也可以试试
python main_amp.py --opt_level O2 --niter 1
# 纯 FP16 精度训练，一般用不上，比较容易遇到 NaN
python main_amp.py --opt_level O3 --niter 1
```

## 小结

这里我们使用 conda 虚拟环境管理我们的 python 开发环境，但是 Anaconda 因为授权原因，无法分发 nvcc 编译器，只能打包 `cudatoolkit` 和 `cudnn`，他们通常只包含 PyTorch、TensorFlow 等深度学习框架运行所需的运行时环境，即 runtime lib．在需要编译一些 PyTorch 的扩展包的时候，我们就需要一个 nvcc．但是我们还是想继续使用 conda 来安装 PyTorch．这个时候只需要使用已经与 conda 虚拟环境中相同版本的 CUDA 对应的 nvcc 即可．

## 补充

如果用户一开始就是用 `pip` 安装的 PyTorch 呢？其实还是一样的，只需要在编译 PyTorch 扩展的时候使用与 PyTorch 一样的 CUDA 版本即可．我们还是首先创建一个 conda 虚拟环境：

```shell
conda create -n apex-pip python=3.7
```

然后在 conda 虚拟环境中安装 PyTorch，按照 PyTorch 官网 [Get started](https://pytorch.org/get-started/locally/) 的提示，选择好版本，系统，包管理器，语言和 CUDA 版本．这里以最新的稳定版的 PyTorch，CUDA 10.0 为例，我们得到安装命令为 `pip3 install torch torchvision`．因此：

```shell
# 激活 conda 虚拟环境
conda activate apex-pip
# 安装
# 这里使用 pip，因为我们这个虚拟环境里只有 python3，pip 默认就是 pip3
# pip3 一般是一个软链接，在这个虚拟环境里是不存在的
pip install torch torchvision
```

接着是安装 apex：

```shell
git clone https://github.com/NVIDIA/apex
cd apex
CUDA_HOME=/usr/loca/cuda-10.0 pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" .
```