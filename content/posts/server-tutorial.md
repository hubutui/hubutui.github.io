---
title: "服务器使用指南"
date: 2017-10-20T00:00:00+08:00
---

## 前言

本指南主要覆盖 Linux 和 Windows 服务器的使用．

## Windows 服务器的远程连接

### 文件传输

关于 Windows 服务器的远程连接，我并不想说太多，只是在这里提一下文件的传输．打开远程桌面连接之前，点开选项，按照下图设置一下：

![](/img/remmina-rdp-driver-map.png)

这样即可以将本地的驱动器（硬盘）映射到服务器上，可以直接在服务器上打开本地的驱动器，方便进行文件的传输．在服务器上看到的大概是这样子的：

![](/img/remmina-rdp-driver-mapped.png)

另外一种传输文件的方式就是使用 FTP 服务了，可以下载安装一个 FTP 客户端，推荐使用 FileZilla，可以直接点击[这里](http://172.21.126.221)下载．安装好 FileZilla 之后，打开，输入服务器 IP，用户名与密码，端口无需填写，直接连接即可．FileZilla 左侧是本地的目录，右侧是远程服务器的目录．上传文件只需在左侧窗格选择文件，右键选择上传；下载文件则是从右侧窗格选择文件或者目录，然后右键选择下载．Windows 服务器上的 FTP 目录在某个盘下的 FTP 文件夹里，因为每台服务器上的硬盘剩余空间不同，所以没能统一都放在同一个盘符下．这个目录应该也不难找到，上传成功后可以根据需要移动到自己的目录下．Linux 服务器也配置好了 FTP 服务，同样可以使用 FileZilla 上传和下载．默认的上传目录为用户的主文件夹，即 `/home/username`，其中 `username` 为用户名．

Windows 默认服务器默认使用 ftp 协议，所以主机地址应该是类似这样子的：

```text
ftp://172.21.126.221
```

Linux 服务器建议使用 sftp 协议，它的主机地址类似这样的：

```text
sftp://172.21.126.221
```

![](/img/filezilla.png)

如果只需要从 FTP 服务器下载的话，也可以直接使用网页浏览器操作．网页浏览器地址栏直接输入 `ftp://IP`，然后输入用户名和密码登录即可．

## Linux 服务器的远程连接

### Remote Desktop Protocol (RDP)

服务器上有配置好的 xrdp 服务，支持直接使用 Windows 的远程桌面连接，直接输入服务器 IP 地址，然后在登录界面输入用户名和密码即可登录．默认的桌面环境为 [Xfce4](https://xfce.org)，与刀片服务器上的是一样的．

需要补充说明的是，RPD 映射的驱动器在远程桌面里的位置为 `~/thinclient_drives/`．

* 优点：可以直接使用 Windows 的远程桌面连接，支持双向剪贴板，支持驱动器映射．
* 缺点：RDP 协议不开源，xrdp 本身也有不少 bug，本身还在不断地开发之中．
* Note: 如果遇到文件管理器无法打开或者 `thinclient_drives` 目录权限错误的问题，你可以想要看看这篇[文章](/posts/little-issue-about-xrdp/)．

### Virtual Network Computing (VNC)

#### 服务端设置

VNC 的话需要进行一些简单的配置．首先输入命令 `vncserver` 启动一个 vncserver session，第一次启动会让你设置一个 vnc 的登录密码．然后它就会新建好一个 vncserver session，并给出类似这样的提示：

```shell
You will require a password to access your desktops.

Password:
Verify:
Would you like to enter a view-only password (y/n)? n

New 'lab-ubuntu-237:1 (matlab)' desktop is lab-ubuntu-237:1

Creating default startup script /home/matlab/.vnc/xstartup
Creating default config /home/matlab/.vnc/config
Starting applications specified in /home/matlab/.vnc/xstartup
Log file is /home/matlab/.vnc/lab-ubuntu-237:1.log
```

注意，密码输入并不会显示有星号的．view-only 密码一般可以不用设置．

不过我们还需要先写一点简单的配置．关闭这个 vncserver session，`vncserver -kill :1`．编辑配置文件 `~/.vnc/xstartup`，修改后该文件的内容为：

```shell
#!/bin/sh
# setup fcitx environment
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
# start xfce4
startxfce4 &
```

默认安装的桌面环境为 [Xfce4](https://xfce.org)．以上这些设置应该就足够了，更多详细的设置可以参考 TigerVNC 的手册．

现在重新启动 vncserver，输入命令 `vncserver` 即可．你应该会看到类似这样的提示：

```shell
New 'lab-ubuntu-237:3 (matlab)' desktop is lab-ubuntu-237:3

Starting applications specified in /home/matlab/.vnc/xstartup
Log file is /home/matlab/.vnc/lab-ubuntu-237:3.log
```

我们只需要关注第一行的最后一个数字，这是 vncserver 的端口．后面使用 VNC client 连接的时候会用到．

#### 客户端设置

本地安装 TigerVNC，打开 TigerVNC 后，输入 VNC server 的 IP 地址和端口，例如 `172.21.125.237:3`，然后点击连接，接着输入之前设置的 vnc 连接密码即可登录．注意，这里的 `3` 就是 vncserver 的端口，也就是前面在服务器端启动 vncserver 的时候，这行

```shell
New 'lab-ubuntu-237:3 (matlab)' desktop is lab-ubuntu-237:3
```

的最后一个数字．当然，实际上它的端口并不是真的是 3，它的端口起始为 5900，所以这里实际上的端口是 5903．所以，实际上你去连接 `172.21.125.237:5903` 应该是一样的效果．有一些 VNC client 就是直接使用的 5903 这样的端口号，而不是自动计算正确的端口，使用的时候稍微注意一下就好了，不是什么大问题．另外说明的一点是，VNC 连接只需要 VNC 密码，所以建议不要设置成与登录密码一样的．TigerVNC 的客户端虽然比较简陋，也有一些选项可以设置，可以自己尝试．这里不再细说．

Windows 下的 VNC 客户端还有很多，比如 TightVNC, UltraVNC, RealVNC, MobaXterm 等，可以根据自己的喜好选择．

#### 另一种设置方式（需要管理员多干活的）

其实还有一种方式，只是管理员要多干活，我就懒得写了．首先为了避免用户注销后 vncserver 进程被杀死，管理员需要为指定用户启用 linger 选项：

```shell
loginctl enable-linger username
```

以用户 matlab 为例，新建一个用户：

```shell
useradd -m matlab
```

设置密码：

```shell
passwd matlab
```

切换到 matlab 用户：

```shell
su matlab
```

以用户模式启动 vncserver 服务：

```shell
systemctl --user start vncserver@:display
```

其中这里的 display 表示第几个 DISPLAY，从 1 开始算，因为第 0 个默认就是物理显示器．配置文件的修改与之前一样，这里不再赘述．然后还可以设置好开机启动服务：

```shell
systemctl --user enable vncserver@:display
```

当然，停止服务的命令为：

```shell
systemctl --user stop vncserver@:display
```

重启服务的命令为：

```shell
systemctl --user restart vncserver@:display
```

查看服务是否正确启动，有的时候 :display 已经被占用了，请更换一个：

```shell
systemctl --user status -l vncserver@:display
```

**注意：**如果你使用这种方式启动 vncserver 的话，请通知管理员执行命令：

```shell
loginctl enable-linger username
```

以免 vncserver 被杀死．
### Secure Shell (SSH)

SSH 连接应该是最方便稳定的连接方式了．不过需要在终端命令行下操作．需要了解一些 Linux 的命令，后文会介绍常用的 Linux 命令．

Windows 下的 SSH client 推荐使用 [Mosh](https://mosh.org)，这个实际上是一个 Chrome App．还有其他的 SSH client，如 [PuTTY](http://www.putty.org), [MobaXterm](https://mobaxterm.mobatek.net)．

这个 MobaXterm 值得推荐一下，直接支持 SSH, RDP, VNC, FTP．看起来还不错，可以考虑使用．

SSH 连接也不难，直接打开 SSH client，输入用户名和密码，端口为默认的 22 端口，一般都不用填写，直接就可以连接．

### 远程桌面连接小结

RDP 协议一开始就是微软自己搞出来的，其他系统对这个协议的支持并不是特别好，难免会有各种问题．但是它的优点在于可以直接使用 Windows 系统自带的远程桌面连接，无需使用第三方客户端．

而 VNC 是大多数 Linux 发行版都会有的，支持良好，是比较靠谱的选择．但是 Windows 下好用的 VNC client 并不是很多．

SSH 是基于终端命令行操作的，不熟悉命令行的话，用起来没有顺手．

个人会优先推荐使用 SSH，其次是 VNC，最后是 RDP．

## Linux 下的常用软件

这里简单介绍一下 Linux 下的常用软件，主要还是给个简单的列表．如果这个列表有一些软件没有列出，你有需要的，也可以安装的．

* 办公套件：WPS Office for Linux, LibreOffice．这个主要是为了方便偶尔需要在 Linux 下查看一些 M$ Office 的文档准备的．
* PDF 文档阅读器：qpdfviewer, FoxitReader．
* 文件管理器：PCManFM File Manager
* 电子词典：GoldentDict．这个只是一个词典程序，本身并不包含任何词典．不过一般都直接用谷歌翻译了吧．
* 图片查看：lximage-qt．
* 终端模拟器：lxterminal．
* 简易文本编辑器：leafpad．这里要提的一点是，Linux 下默认使用 UTF-8 编码，而 Windows 下的文本文件默认使用 GBK 编码，所以有的时候会出现乱码的情况．个人的建议是推荐使用 UTF-8 编码保存文件．
* 压缩与解压：GUI 界面的 Xarchiver，命令行下的 `tar`, `7z`, `unzip`, `unrar`．提到压缩文件就不得不说压缩文件的乱码问题．这个主要出现在 zip 压缩包．zip 本身设计上有问题，并不会存储编码信息，所以 Windows 下创建的带有中文文件名称的 zip 文件，到了 Linux 下解压后会遇到文件名的乱码问题．最简易的解决方案是使用其他的压缩格式，如 7z, rar．
* 代码编辑器：Visual Studio Code 已经安装好了．如果你有需要使用 [Sublime Text](https://www.sublimetext.com/) 或者 [Atom Editor](https://atom.io) 的也可以安装．
* 网页浏览器：Chromium/Chrome, FireFox．
* 文件传输：FileZilla．

### 中文输入

默认安装的输入法为 fcitx-rime 和 fcitx-sunpinyin．默认情况下，fcitx 应该会随桌面环境启动．如果默认的系统语言为英文的时候，启动了 fcitx 之后也没有能直接用．fcitx 本身只是一个输入法框架，需要自己添加输入法．添加方法很简单，右键点击右下角的输入法图标->Configure，点击加号添加输入法，取消 Only Show Current Language 前的勾选，搜索栏输入 rime，找到 Rime 并添加即可．输入法切换快捷键同样是 CTRL + SPACE，fcitx-rime 默认输出是繁体，可以按 `CTRL + `` 选择简体输出．想要进一步了解中州韵输入法设置的，可以去看看它的[官网](http://rime.im)．

### MATLAB 打开 m 文件乱码

Linux 默认使用的编码为 [UTF-8](https://zh.wikipedia.org/wiki/UTF-8)，Windows 默认使用的编码为 [GBK](https://zh.wikipedia.org/wiki/汉字内码扩展规范)，所以会有乱码出现．解决方法有这么几个：

1. m 文件全部使用 UTF-8 编码．
1. 指定 Linux 下 MATLAB 启动时候的环境变量 `LANG` 为 `zh_CN.GBK`．

这里推荐使用第二种方法，Desktop Entry 我已经写好了．确认 m 文件的编码为 GBK 的就选择打开 "MATLAB 2017a - GBK"，否则打开 "MATLAB 2017a - UTF8" 即可．有兴趣看看细节的可以看看文件 `/usr/share/applications/matlab-gbk.desktop` 的内容，大概是这样子的：

```shell
[Desktop Entry]
Type=Application
Version=2017a
Icon=/usr/local/MATLAB/R2017a/toolbox/nnet/nnresource/icons/matlab.png
Name=MATLAB R2017a - GBK
Comment=Start MATLAB - The Language of Technical Computing
Terminal=false
Exec=env LANG=zh_CN.GBK LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6 MATLAB_SHELL=bash /usr/local/MATLAB/R2017a/bin/matlab -desktop
Categories=Development;
```

## 常用 Linux 命令

### Linux 命令基础知识

这里还是先简略介绍一点 Linux 命令的基础知识．首先 Linux 的终端 terminal 是通过 SHELL 与系统进行交互的．常用的 SHELL 有 [bash](https://zh.wikipedia.org/wiki/Bash), [fish](https://zh.wikipedia.org/wiki/Fish) 和 [zsh](https://zh.wikipedia.org/wiki/Z_shell)．大多数 Linux 发行版的默然 SHELL 为 bash．服务器上给各个用户设置的 SHELL 也是默认为 bash．fish 是比较简单易用的 SHELL，有非常友好的命令补全功能．个人非常推荐使用 fish．在终端里输入 `echo $SHELL` 可以查看当前使用的 SHELL．这里 `$` 表示 `SHELL` 是一个环境变量．Linux 下会有很多设置好的环境变量，例如 `PATH` 环境变量里保存的是当前用户的可执行程序的路径．

与 Windows 的 C 盘、D 盘等各种盘符不同，Linux 的文件是一个树形的结构，根目录就是 `/`，其他目录都根目录底下．普通用户的 home 目录一般为 `/home/username`，环境变量 `HOME` 保存的是当前用户的 home 目录．此外比较常见的就是用 `~` 表示 home 目录．还要了解的一个概念就是相对路径与绝对路径，绝对路径就是指从根目录开始写出的完整路径，如 `/usr/bin/fish` 这样子的．相对路径则是相对当前路径来表示的路径，如 `..` 表示上级目录，`.` 表示当前路径．

Linux 命令默认使用空白符作为分隔符，即空格符、换行符和制表符．一般的，如果你使用的命令的参数含有特殊字符，建议使用英文的双引号或者单引号包围起来．值得一提的是，双引号与单引号大多时候作用是一样的，但是也有略微的差别．当你的参数中包含一个变量，也就是要用到 `$variable` 的时候，需要使用双引号；其他时候双引号与单引号没有什么太多区别． 


Linux 下的命令繁多，我们也只需记住常用命令的基本用法和常用的选项．不懂的时候，再输入 `command --help` 或者 `man command` 命令查阅．

### 文件命令

* `ls -lh` 列出当前目录下的文件和目录信息．
* `cd dir` 更改目录到 `dir`．
* `pwd` 显示当前目录．
* `mkdir dir` 创建目录 `dir`．
* `rm file`, 删除文件 `file`．删除目录需要加上 `-r` 选项．这是一个比较危险的命令，建议要确认好之后再支持，否则重要文件瞬间删除了，找回是很难的．
* `trash file` 将文件 `file` 放到回收站，`trash-list` 可以查看回收站里的文件，`trash-restore file` 可以将回收站里的 `file` 恢复，`trash-empty` 则清空回收站．
* `cp file1 file2` 将文件 `file1` 复制到 `file2`．
* `cp -r dir1 dir2` 将目录 `dir1` 复制到 `dir2`，如果 `dir2` 不存在则创建它．
* `mv file1 file2` 将文件 `file1` 复制到 `file2`．
* `mv file dir` 将文件 `file` 复制到目录 `dir` 下．
* `ln -s target link` 创建一个指向 target 的软链接 `link`，比较类似 Windows 下的快捷方式．
* `unlink link` 断开软链接 `link` 与目标的链接．
* `touch file` 创建文件 `file`．
* `cat file` 输出文件 `file` 的内容．
* `head file` 显示文件 `file` 的前 10 行．
* `tail file` 显示文件 `file` 的后 10 行．
* `less file` 显示文件 `file` 的内容，并且可以按 PageUp 和 PageDown 键翻页查看，退出查看请按 q．

### 文件权限

这里并不打算详细介绍 Linux 的文件权限系统，只需知道的是当你运行脚本发现无法运行的时候，应该是少了可执行权限，为一个脚本 `script.sh` 添加可执行权限的命令为 `chmod +x script.sh`．

不建议执行来源不明，自己也不清楚脚本内容的脚本．特别是一些脚本或者命令用了各种你不常见的看不明白的符号的，千万不要好奇地去尝试．虽然普通用户没有权限删除系统的文件，但是若脚本有问题，删除了自己的文件也是很令人难过的．

### 文件搜索

`locate file` 命令应该是最快的，因为它直接从一个数据库中查找文件 `file`．但是数据的更新需要管理员权限，现在服务器上已经设置好了定时任务，大约 6 个小时更新一次数据库．

`find -name path expression` 表示在目录 `path` 下按照文件名查找符合表达式 `expression` 的命令．这个命令还是比较复杂的，个人也很少用．

`tree dir` 将目录 `dir` 的内容以树状形式打印出来．

### 系统信息

* `uptime -p` 显示系统从开机到现在的运行时间．
* `uname -a` 显示内核信息．
* `lscpu` 查看 CPU 信息．
* `free -h` 查看内存以及交换空间（所谓虚拟内存）使用情况．
* `htop` 显示系统运行信息，包括各种进程，CPU 使用情况，内存使用情况等，支持使用鼠标操作．
* `df -lh` 查看系统磁盘占用情况．
* `du -lh` 查看当前目录空间占用情况．
* `nvidia-smi` 查看 GPU 占用情况．

### 压缩与解压

Linux 下常用的压缩包一般就是 tarball，例如 `tar.gz`, `tar.xz`, `tar.bz2` 等．rar, zip, 7z 等格式也有支持．

* `tar cvf dir.tar dir` 将目录 `dir` 打包为 `dir.tar`．
* `tar cvfz dir.tar.gz dir` 将目录 `dir` 打包并使用 gzip 方式压缩为 `dir.tar.gz`．
* `tar xfv tarball` 将 `tarball` 解压到当前目录下．
* `tar tlf tarball` 列出 `tarball` 的文件列表．

其他的压缩格式一般可以使用 [P7ZIP](http://p7zip.sourceforge.net) 解压和压缩．

* `7z x [file.zip or file.rar or file.zip]` 将压缩包解压到当前目录下．
* `7z l [file.zip or file.rar or file.zip]` 列出压缩包的内容．

关于解压需要注意的一点是，请先检查压缩包的文件列表，避免解压后发现文件散落一片，难以处理．实在不行就先建立一个临时文件夹，然后解压到临时文件夹里．

#### 中文文件名乱码

除了之前提到的使用其他压缩格式替代 zip，如果真的不幸遇到解压出来的 zip 压缩包有乱码也还是可以解决的．使用 unzip 解压，并且加上 `-O` 选项即可．

```shell
unzip -O GBK file.zip
```

指定 zip 压缩包中的文件名编码为 GBK 编码．

### 网络与下载

* `ip addr` 查看网卡地址等信息．
* `ping -c 1 www.baidu.com` ping 百度主页一次，检查网络的连通性．
* `wget -c url` 下载 `url` 指向的文件到当前目录，`-c` 选项表示开启断点下载，若中途停止，重新执行命令会从中断的位置继续下载．
* `aria2c -c url` 下载 `url` 指向的文件到当前目录．
* `axel -a url1 url2 url3...` 从多个网址下载同一个文件到当前目录，一般速度会比较快．

此外 `curl` 也可用于下载，但是它的长处在与上传，特别是提交表单．服务器上有一个 `drcom.sh` 脚本，里面就是用的 `curl` 进行校园网的认证登录．如果你没有打开浏览器登录校园网的话，终端下可以使用这个 `drcom.sh` 脚本进行登录．用法：

```shell
drcom.sh username passwd
```

其中的 `username` 为登录帐号，即校园卡号，`passwd` 为登录密码．

### vi/vim 文本编辑器

终端下比较常用的文本编辑器就是 vim 了．输入 `vim file` 即可启动 vim 并打开文件 `file` 进行编辑．vim 主要有三种模式：普通模式，插入模式和命令行模式．

从普通模式进入插入模式可以直接按 i 键，退出插入模式回到普通模式请按 Esc 键，普通模式进入命令行模式请按 SHITF + :．

普通模式主要是用于各种操作，如光标的移动，文本的删除等操作．普通模式常用的操作有：

* 按 i 键在光标位置开始输入．
* 按 I 键在行首开始输入．
* 按 a 键在光标位置后开始输入．
* 按 A 键在行尾开始输入．
* 按 o 键表示在下一行开始输入．
* 按 O 键表示在上一行开始输入．
* 按 x 键表示删除光标位置的字符，这个操作前面可以加上一个数字，表示重复该操作的次数．
* 按 dd 表示删除当前行，同样可以在前面加上一个数字．
* 按 yy 表示将当前行挂起到剪贴板，同样可以在前面加上一个数字．
* 按 p 键表示将剪贴板的内容粘贴．
* 按 u 表示撤销操作．
* 按 CTRL + R 表示撤销之前的撤销操作．
* 按 h, j, k, l 键表示将光标向左、下、上、右方向移动，同样可以在操作前加上一个数字表示重复次数．此外，直接使用方向键移动也是可以的．

命令行模式主要就是用于执行命令，特别是保存文件等操作．

* 按 SHIFT + :，然后输入 `w` 可以保存文件．命令执行完毕后会回到普通模式．
* 输入 `w file` 可以将文件保存为 `file`．
* 输入 `x` 可以保存文件并且退出．
* 输入 `q!` 表示不保存文件，强制退出．

以上这些操作应该足矣覆盖常用的编辑需求，如果需要更多更深入的学习，可以在终端里输入 `vimtutor` 命令，会有一个简单的官方教程教你如何高效地使用 vim．

终端下还有一个比较有名的编辑器就是 [GNU Emacs](https://www.gnu.org/software/emacs/)，有兴趣的可以自己尝试使用．此外，[GNU Nano](https://www.nano-editor.org) >是一个比较简单的编辑器．不想用 vim 这么复杂的，可以尝试使用 Nano，它会明显的操>作提示，使用会比较简单．

### 将程序放在后台执行

有一些操作会耗费较长的时间，一般可以放在后台执行，断开 SSH 连接之后还能够恢复会话．这里只简单介绍 [GNU Screen](https://www.gnu.org/software/screen/)．

直接在终端输入 `screen` 即可新建一个 session，然后在打开的终端 session 中进行操作，断开 session 可以使用快捷键 CTRL + A + D．要恢复一个断开的 session，输入命令 `screen -r` 即可．若有多个断开 session，它会提示你加上 session 的名字，这样就可以恢复指定的 session．

此外还有 [tmux](https://github.com/tmux/tmux/wiki)，想用的可以自己去了解一下．

### 使用 git 进行代码版本管理

[git](https://git-scm.com) 是写出 Linux 内核的大神 [Linus Torvalds](https://zh.wikipedia.org/wiki/林纳斯·托瓦兹) 在不到一个月的时间内写成的代码版本管理工具．现在 git 是比较流行的分布式代码版本控制工具，[GitHub](https://github.com) 应该就是最著名的使用的 git 的代码托管网站了．了解简单的 git 命令还是很有必要的．

* `git clone url` 从 url 克隆一个代码仓库．
* `git init` 在当前目录下新建一个 git 代码仓库．
* `git init project-name` 新建一个目录 `project-name`，并将其初始化为一个代码仓库．
* `git add file` 将文件 `file` 添加到暂存区．
* `git commit -m message` 将暂存区的内容提交到仓库区，`message` 为提交代码时候添加的一个日志．
* `git mv file-orig filr-new` 将文件 `file-orig` 改名为 `file-new`．
* `git branch` 列出所有本地分支．
* `git branch -r` 列出所有远程分支．
* `git branch -a` 列出所有本地分支和远程分支．
* `git checkout -b branch` 新建一个分支 `branch` 并切换到该分支．
* `git checkout branch` 切换到 `branch` 分支．默认的主分支为 `master`，所以 `git branch master` 命令切换到到 `master` 分支．
* `git push` 将已经提交的内容推送到远程仓库．
* `git status` 查看当前状态信息，会有一些提示的命令．
* `git log` 查看 commit 日志．
* `git diff` 查看当前工作去与当前分支最新 commit 之间的差异．
* `git fetch` 从远程仓库下载所有变动的代码．
* `git pull` 从远程仓库下载所有变动的代码，并与本地分支合并．

更多常用的命令可以参考阮一峰写的[这篇文章](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)．

建议注册一个 github 帐号，github 的使用这里不提．github 本身提供了详细的帮助信息．

### 快捷键

CTRL + C 停止当前命令，这个与 MATLAB 类似．所以相应的复制粘贴命令一般是 CTRL + SHIFT + C 和 CTRL + SHIFT + V．CTRL + L 可以清除当前屏幕的内容．

### Unix/Linux 命令速查表

[这里](http://i.linuxtoy.org/files/pdf/fwunixref.pdf)有一份命令速查表，可以收藏一下．内容不多，只有一页，打印出来对照使用也是个不错的选择．

### MATLAB 相关的部分

Linux 下的 MATLAB 默认使用的快捷键集为 Emacs Default Set，可以在 Preferences->Keyboard->shortcuts 里改为 Windows Default set．

MATLAB 中可以直接执行系统的命令，最简单的方法就是先输入一个感叹号，然后输入相应的命令，如 `!ls -lh` 列出当前目录的文件列表．或者使用 `system` 函数．

关于 MATLAB 的启动，默认的启动命令为

```shell
env LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6 \
MATLAB_SHELL=bash /usr/local/MATLAB/R2017a/bin/matlab
```

也就是使用了系统的 `libstdc++` 库，并且指定 MATLAB 使用的 SHELL 为 bash．这个命令已经写入了 `/usr/share/applications/matlab.desktop`．如果你从命令行启动，可以自己修改环境变量．

Ubuntu 上默认的 gcc 版本为 5.4.0，MATLAB 默认支持的版本为 4.9.x，所以会有警告，但是一般不影响使用．而使用 gcc 5 以上版本编译的时候链接的库 `libstdc++` 与之前的 gcc 4.9 完全不同，所以这里需要设置使用系统自带的 `libstdc++`，而不是 MATLAB 自带的 `libstdc++` 库，否则会出错．

## 常用包的安装版本和路径

* MATLAB 2017a, `/usr/local/MATLAB/R2017a`
有两个做好的 desktop 文件：
	- `/usr/share/applications/matlab-gbk.desktop`：GBK 编码，从 Windows 复制过来的 .m 文件中包含中文的时候使用．
	- `/usr/share/applications/matlab-utf8.desktop`：UTF8 编码，Linux 默认使用的编码．
* CUDA 8.0, `/usr/local/cuda-8.0`
	- CUDNN 5.1, `/usr/local/cuda-8.0/cudnn-5`
	- CUDNN 6, `/usr/local/cuda-8.0/cudnn-6`
* Anaconda, `/opt/anaconda3`．如果有需要，可以在自己的目录下安装 Anaconda．自己安装 Anaconda 的，可以使用中科大的开源镜像，速度很快，参考中科大开源镜像的 [Anaconda 源使用帮助](http://mirrors.ustc.edu.cn/help/anaconda.html)
* pycharm-community, `/opt/pycharm-community`．

### Note

`/opt`, `/usr/local` 都不在用户自己的 `$PATH` 环境变量中，有需要的可以自己添加．CUDNN 的路径一般需要添加到 `$LD_LIBRARY_PATH` 环境变量中，`$CUDA_HOME` 环境变量也请自己设置．使用 bash 的用户一般可以添加到 `~/.bashrc` 文件中．

## 后序

本文托管在 [Github Pages](https://hubutui.github.io/post/server-tutorial/)，会不定期更新，请访问该地址获取最新版本．文中提到的软件以及一些其他工具可以从[这里](http://172.21.126.221)（仅限内部网）下载．

有什么需要补充可以跟我提出，也好更新本文．能力有限，难免有写错漏，请各位不吝赐教．
