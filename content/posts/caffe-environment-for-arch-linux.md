---
title: "Arch Linux 下的 Caffe 环境搭建"
date: 2017-09-30T00:00:00+08:00
---

## 简介

这里是搭建 Caffe 环境是为了跑 [HeartSeg](https://github.com/yulequan/HeartSeg) 的代码，里面用到了 [3D-Caffe](https://github.com/yulequan/3D-Caffe.git)，而后者需要编译 Caffe．这就是基本的脉络．

## 下载源码

```shell
git clone https://github.com/yulequan/HeartSeg
cd HeartSeg
git clone https://github.com/yulequan/3D-Caffe.git
```

## 编译 3D-Caffe, 实际上就是编译 Caffe

## 依赖包

根据 Caffe 的 [installation](http://caffe.berkeleyvision.org/installation.html#prerequisites)，需要安装的依赖有：

1. CUDA
	2. version 6.x, 或者 7+
	2. 不建议安装更低的版本
	2. 这里直接安装了 Arch Linux 打包好的 cuda 8.0
1. BLAS，可以选择 ATLAS, MKL 或者 OpenBLAS, 这里安装了 Arch Linux CN 打包的 openblas-git
1. protobuf, 从官方手册上看，当时的版本是 3.0，目前安装最新版本 3.3.2 也没有发现问题
1. glog, gflags, hdf5

### 可选依赖

1. OpenCV >= 2.4, 包括 3.0, 这里安装的是 Arch Linux 自带的 opencv 3.3.0
1. lmdb, leveldb, 注意 leveldb 依赖 snappy
1. python 接口需要 python 以及 numpy 的支持，python 2 或者 python 3 都可以，默认是 python 2, 直接使用默认的即可
1. MATLAB 接口需要有合适的 mex 编译器，目前 MATLAB 2015a 支持的 mex 编译器为 gcc 4.9.x，但是实际上用更高版本的 gcc 也是可以的，目前直接使用 gcc 7.2.0．
1. 如果要用 cuDNN 的话，官方推荐使用 cuDNN 6 或者 cuDNN 7，这里实际没有启用．

### 开始编译啦

```shell
cd 3D-Caffe
cp Makefile.config.example Makefile.config
```

编辑 `Makefile.config`，

```text
# 使用 OpenCV 3
OPENCV_VERSION := 3

# 指定 CUDA 目录，这个目录是 Arch Linux 的安装目录，其他系统自行更改
CUDA_DIR := /opt/cuda

# 使用 openblas，atlas 也可以，但是 Arch Linux 没有打包，AUR 上有，但是编译需要很长时间
BLAS := open

# 指定 MATLAB 目录
# 实际上这就是 matlabroot 目录
# 在 MATLAB 中运行 matlabroot 命令的结果
MATLAB_DIR := /usr/local/MATLAB/R2015a
```

以上各项根据具体情况设置．如果需要使用 cuDNN 的，还要修改这行：

```
USE_CUDNN := 1
```

如果使用 DenseVoxNet 中的模型的话，必须要用 CUDNN．因为开启 CUDNN 后无法编译，暂时没有进一步的测试．

开始编译 3D-Caffe：

```shell
make -j 8
```

编译 MATLAB 接口：

```shell
make -j 8 matcaffe
```

这里可能会提示 `gcc` 版本太高，可以忽略．

## 安装 MATLAB 工具包 NIfTI\_tools

直接从 [NIfTI\_tools](https://www.mathworks.com/matlabcentral/fileexchange/8797-tools-for-nifti-and-analyze-image) 下载工具包，解压后将整个文件夹复制到 MATLAB 的 toolbox 目录，然后打开 MATLAB，设置路径，将以上路径添加，再更新 toolbox 路径即可．

## 运行

1. 运行 code 目录下的 `prepare_h5_data.m`，将 nii 数据转换为 hd5 数据．这个函数可以接受参数，设置输入数据的目录 `data_folder`, 输出数据的目录 `h5_save_folder`, 以及输出数据的列表 `h5_save_list`．
1. 运行 3D-DSN 目录下的 `start_train.sh` 脚本，训练模型．训练参数从 `train.prototxt` 中读取．考虑到 GPU 运算能力与显存的限制，默认的 patch 大小为 64x64x64，但是实际上本机的显存还是不够用，尝试修改 patch 大小为 32x32x32．修改方法是直接修改 `train.prototxt` 中的 `crop_size_l`, `crop_size_h`, `crop_size_w` 的值都改为 32．
1. 训练完成之后就可以运行 code 目录下的 `test_model.m`，进行测试．

### 可能遇到的问题与解决方法

* libstdc++ 版本不兼容的问题：

```text
/usr/local/MATLAB/R2017a/sys/os/glnxa64/libstdc++.so.6: version `GLIBCXX_3.4.21' not found
```

可以在启动 matlab 的时候设置环境变量 LD\_PRELOAD，即

```shell
env LD_PRELOAD=/usr/lib/libstdc++.so.6 matlab
```

这里 env 是 fish 中的用法，bash 的话可能要用 export，详情可以谷歌．

* libharfbuzz.so 缺少特定的 symbol，这个问题在于系统自带的 harfbuzz 和 freetype2 与 MATLAB 自带版本不兼容．同样，直接使用系统自带的 freetype2 即可解决．

```shell
env LD_PRELOAD=/usr/lib/libstdc++.so.6:/usr/lib/libfreetype.so.6 matlab
```

## 参考
* [arch下安装matlab错误小记](http://yinflying.top/2017/07/659#)
* [Caffe Installation](http://caffe.berkeleyvision.org/installation.html)
* [Installation in Ubuntu](http://caffe.berkeleyvision.org/install_apt.html)
* [Installation in Debian](http://caffe.berkeleyvision.org/install_apt_debian.html)

