---
title: "Anaconda 虚拟环境搭建与管理"
date: 2018-11-06T17:21:08+08:00
---

## 安装

直接从官网下载最新版本的 Anaconda3-5.3.0，然直接运行安装即可．Arch Linux 用户可以直接用 ArchLinuxCN 源安装．默认安装路径为 `/opt/anaconda`，使用的时候需要先 `source /opt/anaconda/bin/activate`．

## 添加国内开源镜像

安装完毕第一件事就应该是修改软件源为国内的开源镜像，可以使用中科大或者清华的镜像．Linux 下打开终端，Windows 打开开始菜单->All Programs->Anaconda3(64 bit)->Anaconda Prompt，然后开始输入命令．

中科大的镜像：

```shell
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes
```

或者使用清华大学的镜像：

```shell
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes
```

以上就是我们比较常用的 channel 了，还有其他 channel 可以添加，有需要的同学可以看看开源镜像站点的[说明](https://mirror.tuna.tsinghua.edu.cn/help/anaconda/)．特别是清华大学的开源镜像还提供了 PyTorch 官方 channel，可以根据需要添加：

```shell
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/

# 不再建议使用这个源，除非是兼容旧版代码的需要否则不要使用
# 特此说明，避免直接从网页抄写出错
# for legacy win-64
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/peterjc123/
```



实际上这些命令会创建和修改配置文件 `~/.condarc`，你也可以直接将该文件复制到其他 Linux 服务器上使用．Windows 下这些配置会存储在 `C:\Users\USERNAME\.condarc` 里，其中 `USERNAME` 是你的用户名．

**PS: 这一操作仅需一次，以后该机器上创建的虚拟环境也会使用配置好的开源镜像．**

## 创建和激活环境

创建 conda 虚拟环境只需要键入以下命令：

```shell
conda create --name myenv python=3.6
```

这样就创建好了一个基于 python 3.6 版本的 conda 虚拟环境，虚拟环境中仅含基本的软件，然后你就可以使用以下命令激活该环境：

```shell
conda activate myenv
```

激活该环境之后，即可使用 conda 安装所需的 python 包．比如安装 PyTorch 和 TensorFlow：

```shell
conda install pytorch torchvision tensorflow-gpu
```

conda 会自动帮你安装所需的 cuda．需要注意的是这里 PyTorch 的包名为 pytorch 而非  ~~torch~~．有一些 python 包不存在 Anaconda 的仓库中，此时可以像平常那样使用 pip 命令安装．

**建议：在 conda 虚拟环境中优先使用 conda 命令安装 python 包，conda 不提供的时候再用 pip 安装．**

**不建议：使用 PyCharm 在 conda 虚拟环境中直接添加 python 包是不建议的，会有意想不到的错误．如有兴趣，请耐心等待 PyCharm 的更新修复．**

**提示：**conda 创建的虚拟环境保存在 `~/.conda/envs` 下，在 PyCharm 中设置工程使用的环境的时候，选择 conda 虚拟环境，然后进入该目录查找到对应的 `python` 解释器即可．此外，PyCharm 的 terminal 终端有些 bug，不会自动激活 conda 虚拟环境，但是整个工程还是使用到了 conda 虚拟环境．有需要的可以使用

```shell
source /opt/anaconda3/bin/activate
conda activate myenv
```

来激活环境，以方便在该终端使用．Windows 下的终端就是个残废，没法用的．

如果你不记得了你创建的环境名称，可以使用以下命令来查看：

```shell
conda env list
```

这个命令也会给出 conda 虚拟环境所在的目录．Windows 下 conda 虚拟环境存储的位置最好就用这个来查看了，似乎与 Anaconda 的版本有关，存储位置不确定．

## 环境管理

创建好的环境可以保存下载，然后在其他服务器上创建同样的环境．导出环境：

```shell
conda env export > environment.yml
```

`environment.yml` 文件保存了当前环境中所有的 python 包和对应的版本，将其分享到其他机器即可从该文件创建出一个相同的 conda 虚拟环境．从指定文件创建环境可以使用命令：

```shell
conda create -f environment.yml
```

如果你想要删除一个不再使用的 conda 虚拟环境，可以使用：

```shell
conda env remove --name myenv
```

## 使用 PyCharm 创建 conda 虚拟环境

Linux 的命令行比较好用，Windows 的命令行就比较难用．Linux 下使用 PyCharm 创建 conda 虚拟环境似乎有些莫名其妙的 bug，但是在 Windows 下用命令行创建虚拟环境其实也很难用．总之，使用 PyCharm 创建 conda 虚拟环境都是一个可选项．方法比较简单直观，只需要进入设置，然后创建 conda 虚拟环境即可，

这里注意选择 Conda Environment，然后 Location 是你的虚拟环境保存的地址，建议放到自己的目录下去，Conda executable 是 `conda` 命令的可执行文件的路径，Windows 用户要去 `C:\Anaconda3\Scripts` 或者 `C:\ProgamData\Anaconda3\Scripts` 目录下去找，Linux 用户要去 `/opt/anaconda3/bin` 目录下去找．创建好 conda 虚拟环境之后，注意要使用 conda 进行包管理，否则的话应该是使用 pip 进行包管理．建议使用 conda 进行包管理，否则使用 pip 安装的 tensorflow 和 pytorch 可能会因为无法找到合适的 cuda 而无法工作，conda 会自动安装合适版本的 cuda．

## 后记

以上就是一点点分享，希望能够帮助到大家．conda 命令还有很多详细的用法和选项，个人以为以上这些足以应付我们常用的场景．在 Linux 下建议使用命令行操作，PyCharm 有时候在使用 conda 安装包的时候找不到的话，可以先重启一下 PyCharm，可能是没有正确刷新缓存导致的．

Conda 的 [Managing environments](https://conda.io/docs/user-guide/tasks/manage-environments.html) 强烈推荐大家阅读，内容足够简洁清晰，十几分钟看完即可．其他的章节则可根据自己的需要阅读．