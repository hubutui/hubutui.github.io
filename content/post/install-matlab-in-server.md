---
title: "在 Linux 或者 Windows 服务器上安装部署 MATLAB"
date: 2017-10-03T00:00:00+08:00
---

## 目标

在服务器上安装 MATLAB，以供实验室内多个用户远程连接使用．特别是要完成 MATLAB 的激活．

## 安装方式

这里我们采用的安装方式是这样的，将一台服务器作为 license server，在上面安装 license manager；其他服务器只需能与 license server 连接，也就是能够 ping 通，即可通过 MATLAB 的授权许可．

## 基本步骤

1. 下载 ISO 镜像或者完整的安装包，学校内部网有提供 Windows/Linux/MAC 三合一的完整安装包．
1. 获取用于 license server 的 FIK 和许可证文件．
1. 在 license server 上安装 license manager．
1. 在其他客户机上安装 MATLAB 并激活．

### license server 的安装

#### 准备工作

将作为 license server 的服务器的物理地址和操作系统版本信息发送给许可证管理员，一般就是学校网络中心的老师，请其为 license server 生成 FIK 和许可证文件．

#### 正式安装

启动 MATLAB 安装程序，安装方式选择"User a File Installation Key"，然后按照提示输入 FIK，选择许可证文件，安装产品必选 License Manager．如果该 license server 也要安装 MATLAB 的话，可以将其他组件也选上，也就是全选了．License Manager Configuration 这里要选择 Configure the license manager as a service．最后确认安装．

待安装完成之后，打开 `C:\Program Files\MATLAB\R2017a\etc\win64\lmtools.exe`，`Service/License File` 选项卡，勾选 `LMTOOLS ignore license file path environment vriables`；`Start/Stop/Reread` 选项卡，点击 `Start Server` 即可启动 license manager 服务；`Config Services` 选项卡，`Path to the license file` 这里会显示 `license.dat` 文件的地址，将这个文件复制出来，分发给其他用户在自己的电脑上安装 MATLAB 时使用．最后，在 `Server Status` 选项卡，点击 `Perform Status Enquiry` 可以查看 license server 的运行状态，比如说已经给多少个用户授权等等．此外，在控制面板->管理工具->服务，应该可以看到 MATLAB License Server 已启动．

#### 防火墙设置

打开控制面板->系统和安全->Windows 防火墙->允许的程序，点击更改设置，允许运行另一程序，浏览路径，将 `C:\Program Files\MATLAB\R2017a\etc\win64\lmgrd` 和 `C:\Program Files\MATLAB\R2017a\etc\win64\MLM.exe` 加入允许列表．

至此，license server 的安装与配置就完成了．

### 客户机上的安装

客户机上的安装步骤与上面基本类似，只是不需要安装 License Manager，许可证文件使用的是 license server 提供的 `license.dat` 文件．安装完成之后，应该就可以直接使用了．如果有多个电脑要安装 MATLAB，可以考虑使用非交互式安装方式进行安装．

#### 可能遇到的问题

很有可能遇到的问题是，`license.dat` 文件也安装了，但是打开 MATLAB 会提示无法连接到 license server．但是客户机与 license server 之间可以连接，不存在问题的．实际上这是因为 MATLAB 生成的 `license.dat` 文件有问题，里面默认是使用计算机的 Host 和 MAC 作为标识符，只需改成实际的 license server IP 地址即可．打开供客户机使用的 `license.dat` 文件，将 `SERVER` 行修改为：

```text
SERVER IP INTERNET=IP 27000
```

即可，其中的 IP 为 license server 的 IP 地址．

#### 非交互式安装的配置文件

解压或者挂载下载好的 MATLAB 完整安装包．进入该目录，复制 `installer-input.txt` 文件为 `linux.txt`．然后根据该文件的提示进行编辑．需要注意的是里面的路径要写绝对路径，不能写相对路径．编辑后大概是这样子：

```text
# 安装路径
destinationFolder=/usr/local/MATLAB/R2017a

# FIK 序列号
fileInstallationKey=

# 同意协议
agreeToLicense=yes

# 安装输出的 log
# 方便出问题的时候进行排查
outputFile=

# 安装模式
mode=silent

# 许可证文件
licensePath=
```

对于 Windows 系统，只需根据提示修改一下配置文件即可，配置文件中的说明很详细了，这里不在赘述．

#### 进行安装

只需要一行命令

```shell
sudo ./install -inputFile linux.txt
```

记得将上面 `linux.txt` 换成你自己的配置文件，并且必须使用绝对路径．

Windows 的话应该用的是：

```text
setup.exe -inputFile windows.txt
```

#### Linux 下生成 Desktop Entry

Linux 下没有提供 Desktop Entry，可以自己写一个，MATLAB Logo 可以从[维基百科](https://upload.wikimedia.org/wikipedia/commons/2/21/Matlab_Logo.png)下载，作为图标．

```text
[Desktop Entry]
Type=Application
Icon=/usr/share/icons/matlab.png
Name=MATLAB R2017a
Comment=Start MATLAB - The Language of Technical Computing
Exec=/usr/local/MATLAB/R2017a/bin/matlab -desktop

# 一般来说用上面这行就足够了
# 下面这行是设置 MATLAB 默认使用系统自带的 libstdc++ 和 libfreetype 库，并且指定 SHELL 为 bash
# 而 optirun 是 bumblebee 提供的，如果你安装了 bumblebee 的话
# 实际上台式机是没有必要使用 bumblebee 的，直接使用独显即可，不需要进行核显和独显的切换
# 毕竟谁会在乎那点电量呢？
# 如果你用 bumblebee 的话，要确认接对主机箱后的显示器连线
# Exec=env LD_PRELOAD=/usr/lib/libstdc++.so.6:/usr/lib/libfreetype.so.6 MATLAB_SHELL=bash optirun /usr/local/MATLAB/R2017a/bin/matlab -desktop
Categories=Development;
```

#### 命令别名或者添加环境变量

考虑到 MATLAB 是安装到了 `/usr/local/MATLAB/R2017a` 目录下，默认情况下这个目录并不在 `PATH` 环境变量中，你可以将其添加到环境变量中．一般就是编辑 `.bashrc` 之类的配置文件：

```shell
export PATH=$PATH:/usr/local/MATLAB/R2017a/bin
```

然后记得 `source .bashrc` 使其生效．其他非 bash 的 SHELL 要如何设置环境变量，请参考各自的手册．

另外，也可以设置一个命令别名．或者做一个软链接，方法自行 google．

### 可能遇到的蛋疼问题

一个比较常见的问题就是 `/tmp` 空间不足，导致安装过程的一些压缩包解压失败，错误信息类似这样：

```text
The following error was detected while installing 3p/hdfeos2_maci64:
archive is not a ZIP archive
```

解决方法就是清除一下 `/tmp` 下的文件，或者调整 `/tmp` 的大小．

另外一个比较蛋疼的问题是，安装脚本 `install` 没有可执行权限，即使 `install` 脚本添加了可执行权限，也还是无法安装．这个实际上是 MathWorks 的锅，他们自己打包的 ISO 文件中没有给相应的文件设置好可执行权限．比较简单地坚决方法就是将 MATLAB 的 ISO 镜像解压到文件夹 `matlab`，然后给 `matlab` 的所有文件添加可执行权限：

```shell
chmod +x -R matlab
```

我知道这不是一个很好的办法，毕竟理论上不是所有的文件都应该添加可执行权限．只是这样子比较省事啊．

### Arch Linux 的 AUR 包

[AUR](https://aur.archlinux.org/packages/matlab/) 上有现成的 `PKGBUILD`，可以用来打包．我们需要准备三个文件，`matlab.tar`, `matlab.fik`, `license.dat`．其中，`matlab.fik` 文件中包含 FIK 序列号，`license.dat` 就是上文提到的许可证文件．而 `matlab.tar` 中包含 MATLAB ISO 镜像文件中的所有内容，需要说明的是，我们需要直接将 ISO 镜像文件中的文件直接打包成 `matlab.tar`．

它还很贴心的帮我们生成了 desktop 文件，里面还有其他暖心的修改，详情参考 PKGBUILD 文件中的注释．不过美中不足的是，它的默认安装路径为 `/opt/tmw/matlab`，很奇怪的路径，感觉还是用 `/opt/matlab` 或者 `/usr/local/matlab` 这样的路径比较符合我的口味．此外安装脚本的输出也被重定向到了 `/dev/null`，所以打包失败也不知道哪里失败了．

总之我自己就修改了一下，新的 PKGBUILD 文件在[这里](https://gist.github.com/hubutui/612a10a2a20c7bf6a7e3744f6ac27e5e)．

## 参考

* MATLAB 官方文档：[静默安装](https://cn.mathworks.com/help/install/ug/install-noninteractively-silent-installation.html)
* [如何获取 File Installation Key](https://cn.mathworks.com/matlabcentral/answers/102845-where-can-i-find-the-activation-key-or-file-installation-key-fik-for-my-license)
* MATLAB 官方文档：[无网络条件下的安装](https://cn.mathworks.com/help/install/ug/install-and-activate-without-an-internet-connection.html)
* [同一台电脑为多个用户授权许可](https://cn.mathworks.com/matlabcentral/answers/96008-how-can-i-install-multiple-individual-licenses-on-a-single-computer)
* Ubuntu 的 [document](https://help.ubuntu.com/community/MATLAB)
* 台湾国立中央大学的 [network.lic](http://matlab.math.ncu.edu.tw/download.html)

