---
title: "使用 conda 管理和维护 R 语言环境"
date: 2020-11-07T00:00:00+08:00
draft: false
---

# 前言

Anaconda 是一个免费开源的 Python 和 R 语言的发行版本，主要用于计算科学．Anaconda 使用 conda 作为包管理工具．所以，自然而然的，R 语言用户也可以使用 conda 来管理和维护我们的环境．

# 方法

传统上，如果我们使用 R 语言，我们一般需要自己编译安装大量的包，主要是在 R 的命令行内使用

```R
install.packages("pkgname")
```

R 语言用户比较熟悉的应该是 [CRAN](https://cran.r-project.org/)，它有点类似于 Python 的 [PyPI](https://pypi.org/)，提供了大量的 R 语言的包．但是，编译安装 R 语言包并不是一件轻而易举的事情，特别是当你的 R 包需要系统提供一些依赖的时候．

实际上，如果我们能用 conda 来创建和管理 R 语言的环境，其实会更加轻松很多．Anaconda 中的 R 语言包主要由 conda-forge, r, bioconda 三个 channel 提供．其中 r channel 提供的 R 版本目前最新只有 3.6，需要使用 4.0 版本的建议使用 conda-forge．首先，我们创建一个 R 语言的虚拟环境：

```shell
conda create -n r-env -c conda-forge
```

对于需要安装的 R 包，我们大部分使用可以通过以下命令来安装：

```shell
conda install -c conda-forge r-pkgname
```

其中 `pkgname` 是 R 包在 CRAN 里的名字．一般用户可以先去 anaconda 网页上搜索看是否有所需的包．如果 anaconda 的源没有提供这个包，我们还可以使用传统的方法，即启动 R 命令行，然后使用 `install.packages("pkgname")` 来安装，它会安装到你的 conda 环境目录下．如果有疑问，你可以检查一下 `libPaths`，在 R 命令行中使用：

```R
.libPaths()
```

查看当前的安装目录是否正确．建议不要混合使用 conda 虚拟环境＋自己安装的 R ＋操作系统安装的 R，否则会比较容易出现莫名其妙的错误．

# 参考

https://docs.anaconda.com/anaconda/user-guide/tasks/using-r-language/