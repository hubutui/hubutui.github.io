---
title: "面向深度学习训练的LXD GPU 服务器搭建、管理与使用"
date: 2019-09-09T00:00:00+08:00
toc: true
---

## 目标

1. 管理员除了在前期准备合适的 LXD 容器模板这一步的工作量较大，日常的维护和管理都会更加轻松．
2. 用户在自己的 LXD 容器内拥有完整的权限，可以自己安装所需的各种软件，但是不会影响到其他用户．

以下章节的内容，管理员需要根据实际情况修改"For 用户"此章节的内容，用户一般应该只需要阅读该章节即可．如果继任的管理员可以只阅读"For 管理员"此章节的内容，足以应付日常的管理．而一切从头开始的管理员请完整阅读本文内容．

## For 用户

LXD 容器提供：

1. Ubuntu 18.04 + LXDE 桌面
   * 默认开启 xrdp 服务，可以直接使用 **Windows 远程桌面**连接，默认桌面为 LXDE．默认在用户名为 `ubuntu`，密码由管理员在创建该容器后提供，请根据需要修改，切勿使用默认的弱密码．
   * 默认设置好 VNC 服务，未有设置自动启动．用户可以根据需要使用 `systemctl --user start/status/stop/restart/enable/disable vncserver@:port` 等命令管理该 systemd 服务，注意替换 `port` 为你想用的端口即可．
2. Nvidia 显卡驱动，不含 CUDA．
3. PyCharm 社区版，如有需要使用专业版的请自行安装（专业版可以用学校邮箱去官网注册免费获取，免费使用一年，到期未毕业可免费续期）．
4. 安装在 `/home/ubuntu` 下的 Miniconda．
5. 为了减小容器的大小，容器内 `/usr/local/MATLAB` 目录下的 matlab 挂载自宿主机．
6. 切换 `gcc/g++` 版本可以使用 `update-alternatives --config gcc`．

注意：

1. LXD 容器内未有提供 CUDA，但是 Anaconda/Miniconda 可以提供 cudatoolkit runtime，不含用于编译的 nvcc 编译器，一般足以满足日常开发使用．如需 nvcc 请联系管理员添加一个 cuda 的配置文件．目前只提供 CUDA 9.0 和 CUDA 10.0 两个版本．不同版本的 CUDA 需要搭配指定版本的编译器，请使用 `update-alternatives --config gcc` 切换到所需的版本．
2.  每个容器都分配有独立 IP，校园网范围可直接访问．需要进行校园网用户认证连接外网的可以打开浏览使用网页认证．用户可以在 http://x.x.x.x:38080 上查询每一台宿主机上运行这的 LXD 的详细信息特别是 IP．如果其他疑问，请联系管理员．
3. 用户拥有容器内的全部控制权限，可以根据需要安装和修改容器内的一切东西，甚至可以自己根据需要重启容器内的系统而不影响其他用户的使用．比如安装各种版本的 cuda，更换桌面环境，安装其他软件，等等．在容器内的使用仍可参考群文件中的服务器使用指南，区别在于此时用户有最高权限．
5. 用户在使用管理员提供的密码通过 Windows 远程桌面登录到自己的容器内的系统之后，必须修改密码．修改 VNC 密码请使用命令 `vncpasswd`，然后按照提示修改．修改用户密码请输入 `passwd`，然后按照提示修改．
6. 使用 XRDP 的用户如果遇到 `/home/ubuntu/thinclient_drives` 无法正确挂载的问题，可以使用命令 `fusermount -u /home/ubuntu/thinclient_drives` 将其正确卸载即可．
7. 容器内运行 `nvidia-smi` 只能看到 GPU 显存的使用情况，无法看到具体是哪一个进程在使用．这个是 Linux 内核 namespace 的限制，无解．
8. 用户的数据和代码等应该保存到 `/home/ubuntu/username`，其中 `username` 是你们的用户名，一般为姓名拼音小写．可以使用命令 `df -h` 查看磁盘信息：

```shell
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/loop3      120G   64G   54G  55% /
   /dev/sdc1       459G  175G  261G  41% /usr/local/MATLAB
   /dev/md126      7.0T  2.0T  5.0T  29% /home/ubuntu/tom
```

其中根分区 `/` 一般是在固态硬盘上的，读写速度较快，因而可以将训练过程的数据读写放在 `/home/ubuntu` 目录下；`/usr/local/MATLAB` 是挂载自宿主机的只读目录，容器内的用户无法修改；`/home/ubuntu/tom` 是留给容器内用户用作数据存储读写的，一般是挂载自宿主机的机械硬盘，用户的数据应该保存到此处，代码可以考虑托管到 [GitLab](http://172.21.109.102:30000) 上．

## For 管理员

### 容器管理

使用模板给新用户创建容器：

```shell
lxc launch template container-name
```

其中 `template` 是镜像模板的名称，可以用 `lxc image list` 常看，`container-name` 为用户的容器名称，可以使用 `username-container` 以便识别和管理．LXD 容器的存储池可能容量不足，可以把宿主机上的 `/home/username` 挂载到容器内的 `/home/ubuntu/username` 目录给用户使用．

目前在服务器上提供了两个模板，带桌面的 `template-ubuntu-18.04-desktop` 和不带桌面的 `template-ubuntu-18.04-no-desktop`．带桌面的镜像大小大约 1200M，不带桌面的镜像不到 500M．主要是因为带桌面的镜像还额外安装了浏览器、PyCharm 等需要桌面才能使用的软件．如果有需要，可以从这两个镜像创建容器，修改，然后保存为新的镜像作为模板使用．

举例来说，我们给用户 tom 创建容器，可以这么做：

```shell
# 在宿主机创建用户 tom，并设置其 SHELL 为 nologin，禁止其登录宿主机
useradd -m -s $(which nologin) tom
# 将宿主机中用户 tom 的 home 目录挂载到容器内的 /home/ubuntu/tom 目录去
lxc config device add tom-container home disk source=/home/tom path=/home/ubuntu/tom
# 设置该容器的 idmap，让宿主机的 tom 用户可以映射到容器内的 ubuntu 用户
printf "uid $(id tom -u) 1000\ngid $(id tom -g) 1000" | lxc config set tom-container raw.idmap -
# 检查 raw.idmap 的设置
lxc config get tom-container raw.idmap
# 应该得到类似这样的输出：
uid 1003 1000
gid 1003 1000
# 这说明，宿主机的 tom 用户的 uid 和 gid 都分别映射到容器内的 ubuntu 用户的 uid 和 gid 了
# 也就是说，容器内 ubuntu 用户的权限等价与宿主机里 tom 用户的权限，从而可以读写挂载进容器的
# tom 用户的 home 目录
#
# 最后要重启容器才能使其生效
lxc restart tom-container
```

容器内默认的普通用户的用户名为 `ubuntu`，默认在 `5901` 上开启 VNC 服务．从安全角度考虑，管理员应当修改对应的密码为随机密码，提供给用户，要求用户自行修改：

```shell
# 进入容器
lxc exec container-name bash
# 在容器内修改 ubuntu 用户的密码
# 按提示修改即可，无回显
passwd ubuntu
# 切换到 ubuntu 用户，然后修改 VNC 密码
su ubuntu
vncpasswd
# 修改完毕，退出容器
exit
```

使用命令 `lxc list` 查看容器信息，将其 IP 及帐号信息提供给用户即可．因为用户有用该容器内的所有控制权，所以管理员将容器的信息交付用户之后，一切就由用户自己主宰了．如果有出现任何错误，最简单的方法就是删除现有容器，新建一个即可使用．

对于需要使用 CUDA 中的 `nvcc` 的用户，可以在其容器添加配置文件 `cuda`：

```shell
lxc profile add tom-container cuda
```

**TIPS:** 

1. 用于管理容器的教程到此结束，后面是一步一步从安装 LXD 开始的配置．
2. 部分代码可能会占用过高的 CPU，默认的配置是限制 CPU 核心数量为全部核心数的 3/4，也可以针对指定容器限制其允许使用的 CPU 核心数：

```shell
lxc config container-name set limits.cpu 16
```

### 在不同服务器之间迁移 LXD 容器

其实这个比较简单，只需要使用命令 `lxc copy server01:container01 server02:container02 --stateless --container-only –-verbose` 即可将 server01 上的容器 container01 迁移到 server02 上的 container02．建议使用 `--container-only`，不用复制容器的快照，节省时间；使用 `–-stateless` 通常也会比较稳定一些．

此外，还有一些需要注意的地方，`lxc copy` 还会将 `lxc config` 给该容器设置的配置也复制过来．这可能会有一些意外的结果，需要特别处理一下．比如，你用 `lxc config` 添加了一个磁盘设备，而在 server02 上不存在该设备，则会导致迁移好的容器无法启动，在新版本的 lxd 上甚至是直接无法迁移．对于通过 `lxc config device add` 添加的磁盘设备，可以考虑在 server01 上先将其移除，然后再迁移．很显然，该磁盘中的数据是不会随之迁移，如果有需要，用户还需要自行同步这些数据．其他使用 `lxc config` 命令设置的配置项也应该需要在新的 LXD 容器上重新配置，如 `raw.idmap` 等．

## 从安装 LXD 开始

### 准备

1. 宿主机安装 Ubuntu 18.04 或者 Ubuntu 16.04（建议使用 Ubuntu 18.04），根分区或者用于 LXD 存储池的分区的文件系统为 btrfs 或者 zfs．如果是已经安装好的系统，根分区为其他文件系统，那么 LXD 会创建一个 loopback 设备，IO 性能有所下降．最好是有另外一块硬盘格式化为 btrfs 分区来使用，这也是官方所推荐的．
2. 宿主机安装好最新的 Nvidia 显卡驱动（再次建议使用 Ubuntu 18.04，可以直接从官方源安装 nvidia 430 驱动，足以支持到目前最新的 CUDA 10.1）．Ubuntu 16.04 的驱动安装请参考 https://launchpad.net/~graphics-drivers/+archive/ubuntu/ppa．
3. 为了能在 LXD 容器中使用 GPU，还需要参考 https://github.com/NVIDIA/nvidia-container-runtime 安装 nvidia-container-runtime．
4. 宿主机上的 CUDA 可以安装，也可以不安装．如果你想节约磁盘空间，可以考虑在宿主机安装多个版本的 CUDA，然后挂载到容器内给用户使用．Anaconda/Miniconda 可以提供 CUDA runtime，但是不包含 nvcc，用户一般安装 CUDA 还是因为需要这个，其他的时候用 CUDA runtime 即可．
5. 在容器内完整安装 Anaconda 就太大了，而且如果不用 Anaconda 中默认的 python 版本的时候，自带的很多包都用不上，而是在需要的时候从网络下载安装．因此，可以考虑安装 Miniconda 替代．另外，如果在宿主机安装 Anaconda/Miniconda，然后挂载到 LXD 容器内给用户使用也是很好的解决方法．

### 安装与配置 LXD

对于 Ubuntu 18.04，可以直接安装：

```shell
sudo apt install lxd lxd-client
```

Ubuntu 16.04 使用以下命令安装：

```shell
sudo apt install -t xenial-backports lxd lxd-client
```

首先管理员需要将自己加入 lxd 组：

```shell
sudo gpasswd -a $(whoami) lxd
```

重新登录后生效，然后开始配置：

```shell
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (btrfs, dir, lvm) [default=btrfs]: 
Create a new BTRFS pool? (yes/no) [default=yes]: 
Would you like to use an existing block device? (yes/no) [default=no]: 
# 这里我们把默认的存储池设置为 128G，应该足够使用
# 如果根分区的文件系统是 btrfs/zfs，这里会直接跳过，不会有设置．
# 即默认可以使用根分区的全部容量．
# 其他文件系统的话，这里要创建 loopback 设备，所以才需要指定大小．
Size in GB of the new loop device (1GB minimum) [default=91GB]: 128
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
# 网络设置这部分，我们不适用默认的网桥，选 no
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
# 这里选 yes
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: yes
# 然后输入已有的网络设备名称，可以用 ip addr 命令查看
Name of the existing bridge or host interface: eno1
# 这里设置允许通过网络管理 LXD，如果不需要可以选 no
Would you like LXD to be available over the network? (yes/no) [default=no]: yes
Address to bind LXD to (not including port) [default=all]: 
Port to bind LXD to [default=8443]: 
# 密码不设置，直接回车，我们使用证书来认证
Trust password for new clients: 
Again: 
No password set, client certificates will have to be manually trusted.Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
# 最后打印输出配置的内容，以便检查是否有误
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: yes
config:
  core.https_address: '[::]:8443'
  core.trust_password: ""
networks: []
storage_pools:
- config:
    size: 128GB
  description: ""
  name: default
  driver: btrfs
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      nictype: macvlan
      parent: eno1
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
cluster: null

```
可以看到，默认的配置文件中添加了一个 macvlan 类型的网络设备 eth0，这样就可以给每个容器分配一个校园网网段的 IP．

另外，我们还要配置 idmap，首先查看 `/etc/subuid` 和 `/etc/subgid` 这两个文件的内容，两个文件内容应该是一样的，默认为：

```text
lxd:100000:65536
root:100000:65536
```

这表示将允许用户 `root` 使用宿主机中的 uid/gid，从 100000 开始的 65536 个 id，用于容器内的分配．lxd 中一行按照文档的说法是用于在卸载 LXD 是应该删掉哪些行而已，因此应该与 root 那行保持一致．这样子的话，容器内的 root 用户（uid/gid 均为 0）在宿主机看来他的 uid/gid 就是 100000，容器内的其他用户 uid/gid 也相应的递增映射．也就是说，容器内的 root 用户与容器外的 root 用户并不等价，而是映射到了一个不存在的用户的 uid/gid，规避了安全问题．这是 LXD 的默认运行方式，即在非特权模式下运行．容器内的用户无法逃逸到宿主机，对宿主机造成破坏．但是呢，我们想要让宿主机里的用户 tom 和容器中的 ubuntu 用户相互映射，从而实现宿主机和容器的文件共享，并保证宿主机中的文件权限不会混乱．此时就可以使用 LXD 的 `raw.id` 配置了．为此，将 `/etc/subuid` 和 `/etc/subgid` 文件中的 `lxd` 和 `root` 两行都删掉，重新添加以下四行：

```text
lxd:1000:1000
root:1000:1000
lxd:100000:10000000
root:100000:10000000
```

这里我们把原来允许使用的 id 数量增加了，这是为了嵌套使用容器而设置的，POSIX 要求可用的 id 至少有 65536 个．然后，我们还设置允许用户 root 使用 uid/gid 从 1000 开始的 1000 个 id．这是正好我们在宿主机创建的 uid/gid 默认是从 1000 开始递增的．修改 `/etc/subuid` 和 `/etc/subgid` 需要重启 `lxd` 服务才能生效：`systemctl restart lxd`．

**TIPS：**这里的 idmap 的设置是为了方便管理，管理员也可以考虑跳过．只需要给不同的容器在宿主机上分配不同的目录挂载进去给用户使用，不要允许用户访问宿主机，问题也不大．不过好像这样子管理员在宿主机上也看不出来到底是哪个用户在占用显存了．在宿主机上用 `nvidia-smi` 命令查到进程的 pid，再根据 pid 去查看对应的用户的 uid 应该就都是一样的．为了管理方便，还是多做点吧．

接着添加设备，我们把宿主机上的 MATLAB 和 GPU 添加．`lxd profile` 和 `lxd config` 两个命令的法部分配置是一样的，不过 profile 是针对一系列容器的，可以给多个容器都使用同样的 profile，而 config 则是针对容器的．因此，我们这里把所有容器都共用的部分用 profile 来配置，其他的用 config 来配置．

```shell
# 设置共享的 MATLAB 目录
lxc profile device add default matlab disk source=/usr/local/MATLAB path=/usr/local/MATLAB
# 添加 GPU
lxc profile device add default gpu gpu
# 配置 nvidia.runtime，让容器使用宿主机的驱动和相关 runtime
# 不需要在容器内安装驱动
lxc profile set default nvidia.runtime true
# 开机自动启动容器
lxc profile set default boot.autostart true
# 限制容器可以使用的 CPU 核心数，请根据需要修改
# 不同的服务器的 CPU 不同，这里按照 3/4 的比例来设置允许容器使用的 CPU 核心数量
lxc profile set default limits.cpu 24
# 可以再次检查这个默认的 profile 是否有误
lxc profile show default
```

清华大学开源镜像站点提供 LXC Images 镜像加速，将其添加：

```shell
lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public
```

这个镜像源应该也就第一次创建容器，根据容器制作模板的时候需要，其他的时候应该用不上了吧．

### 创建一个容器

下面创建一个容器，并以该容器作为基础，保存为镜像模板，对于每个新用户都可以从该镜像模板直接创建容器提供给用户使用．管理员的工作负担主要在前期安装 LXD 已经准备合适的镜像模板．

```shell
# 创建容器 demo，基础镜像为 ubuntu 18.04
lxc launch tuna-images:ubuntu/18.04 demo
# 进入容器
lxc exec demo bash
```

按照 https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/ 的提示修改软件源．

```shell
# 在容器内，安装一些必备软件
# 这里根据需要增减
apt update
apt install filezilla tigervnc-standalone-server fcitx-sunpinyin language-pack-zh-hans autocutsel
# 安装编译器和一些常用的编译工具
apt install build-essential gcc-5 gcc-6 gcc-7 gcc-8 g++-5 g++-6 g++-7 g++-8
# 设置 gcc/g++ 编译器的优先顺序，默认使用 gcc-7/g++-7
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 100 --slave /usr/bin/g++ g++ /usr/bin/g++-7
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 80 --slave /usr/bin/g++ g++ /usr/bin/g++-8
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-6 60 --slave /usr/bin/g++ g++ /usr/bin/g++-6
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 50 --slave /usr/bin/g++ g++ /usr/bin/g++-5
# 用户需要切换默认的编译器的时候只需要使用命令
update-alternatives --config gcc
```

安装桌面，根据需要使用以下命令：

```shell
# LXDE 桌面
apt install lxde
# XFCE4 桌面
apt install xfce4
# KDE 桌面
apt install plasma-workspace
# GNOME 桌面
apt install --no-install-recommends vanilla-gnome-desktop
# 还有其他桌面可以安装使用，有需要自己安装
# 建议不要全部安装，太浪费了，装自己需要的桌面即可
```

另外再设置一下 vnc，即准备一个 `/home/ubuntu/.vnc/xstartup` 脚本，内容为：

```text
#!/bin/bash
fcitx-autostart &
export LANG=zh_CN.UTF-8
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
# set clipboard and start desktop environment
autocutsel -fork &
# default DE is lxde
exec startlxde
```

记得加上可执行权限，`chmod +x /home/ubuntu/.vnc/xstartup`．再准备一个 vncserver 的 systemd 服务文件 `/usr/lib/systemd/user/vncserver@.service`，用于开机自动启动 VNC，内容如下：

```text
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=forking
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver %i
ExecStop=/usr/bin/vncserver -kill %i

[Install]
WantedBy=default.target
```

如果有需要的话，再设置一下开机自动启动：

```shell
loginctl enable-linger ubuntu
# 切换到 ubuntu 用户操作
su ubuntu
systemctl --user enable vncserver@:1
```

不要忘记设置 vnc 密码，否则该服务无法启动：

```shel
# 按提示设置一个默认密码
vncpasswd
```

安装 xrdp：

```shell
apt install xrdp
```

配置比较简单，直接用于启动桌面的命令写入到 `/home/ubuntu/.xsession` 即可，一般地使用以下其中一个用于启动对应的桌面即可：

```shell
#startlxde
startxfce4
#startkde
#gnome-session
```

可以考虑配置一下 `/etc/xrdp/xrdp.ini` 文件，我们只使用其中的 xrdpxorg 来连接．因此，配置文件中的 `Session types` 只保留 `Xorg` 和 `Xvnc` 节即可，并且进一步修改为：

```text
[Xorg]
name=Xorg
lib=libxup.so
username=askubuntu
password=ask
ip=127.0.0.1
port=-1
code=20

[Xvnc]
name=Xvnc
lib=libvnc.so
username=askubuntu
password=ask
ip=127.0.0.1
port=-1
```

也就是修改默认的登录名为容器内的用户名 `ubuntu`，当然用户也还可以进一步修改，然后密码就是请求用户输入，更加精简方便．用户直接用 Windows 远程桌面连接即可．这两种方式都可以使用，`Xorg` 就是使用 `xorgxrdp` 模块来实现的 RDP 服务协议，而 `Xvnc` 这个的话就是 `xrdp` 在远程主机上启动一个 VNC 服务，然后允许用户使用 Windows 远程桌面连接，xrdp 负责其中 RDP 协议与 VNC 协议的转换．

安装 Anaconda，`su ubuntu` 切换到 `ubuntu` 用户身份去安装，按照 https://mirror.tuna.tsinghua.edu.cn/help/anaconda/ 的提示从 https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/ 下载最新版本的 anaconda 进行安装即可．安装完毕之后按照该网页的提示添加清华大学的 conda 源．为了节省空间，可以考虑安装 Miniconda．还可以考虑安装到宿主机，然后挂载到容器内．这里不赘述了，参考前面的命令你应该很容易做到的．

安装 PyCharm，直接从[官网](https://www.jetbrains.com/pycharm/download/#section=linux)下载，然后解压复制到合适的目录即可，比如复制到 `/opt` 目录去．有需要的可以写个 `.desktop` 文件，方便使用．当然，也可以把 PyCharm 安装到宿主机，然后像使用 MATLAB 那样直接挂载进容器的指定目录．

### 保存容器为镜像模板

```shell
# 删除 apt 缓存，减小镜像大小
rm -rfv /var/lib/apt/lists/*
# 退出并停止容器
exit
lxc stop demo
# 保存容器为镜像模板
lxc publish demo --alias template-ubuntu-18.04
```

这里我们将容器保存为镜像模板，并且设置该镜像的别名为 `template-ubuntu-18.04`．以后，只需从该模板创建容器给用户使用即可．例如，给用户 tom 创建一个容器，只需要使用命令：

```shell
lxc launch tempate-ubuntu-18.04 tom-container
```

从方便管理的角度考虑，我们在宿主机也创建用户 tom，但是不设置密码同时将其 SHELL 设置为 `nologin`，tom 用户就无法登录到宿主机．同时我们把宿主机 tom 用户的 home 目录挂载到容器内，并设置好权限，使得宿主机中的用户 tom 与容器内的用户 ubuntu 映射起来，从而实现容器内的用户 ubuntu 可以读写挂载进来的 tom 的 home 目录．这样即可以保证容器内的用户可以将数据持久化存储到宿主机挂载进来的目录，而在宿主机这边管理员也可以很好的管理和区分不同用户的文件．

```shell
# 在宿主机创建用户 tom
useradd -m -s $(which nologin) tom
# 将宿主机中用户 tom 的 home 目录挂载到容器内的 /home/ubuntu/tom 目录去
# 如果你是在宿主机安装的 Anaconda/Miniconda，可以直接将 tom 目录挂载到 /home/ubuntu 目录去
# 这样子用户的所有数据都会保存到宿主机上的 /hom/tom 目录，容器出错直接删除即可，数据也不会丢失．
lxc config device add tom-container home disk source=/home/tom path=/home/ubuntu/tom
# 设置该容器的 idmap，让宿主机的 tom 用户可以映射到容器内的 ubuntu 用户
printf "uid $(id tom -u) 1000\ngid $(id tom -g) 1000" | lxc config set tom-container raw.idmap -
# 检查 raw.idmap 的设置
lxc config get tom-container raw.idmap
# 应该得到类似这样的输出：
uid 1003 1000
gid 1003 1000
# 这说明，宿主机的 tom 用户的 uid 和 gid 都分别映射到容器内的 ubuntu 用户的 uid 和 gid 了
# 也就是说，容器内 ubuntu 用户的权限等价与宿主机里 tom 用户的权限，从而可以读写挂载进容器的
# tom 用户的 home 目录
```

### 添加一个 CUDA 的配置文件

```shell
# 创建名为 cuda 的配置文件
lxc profile create cuda
# 挂载 cuda，不同版本都挂载上
# 这里提供了 CUDA 9.0 和 10.0
lxc profile device add cuda cuda-10.0 disk source=/usr/local/cuda-10.0 path=/usr/local/cuda-10.0
lxc profile device add cuda cuda-9.0 disk source=/usr/local/cuda-9.0 path=/usr/local/cuda-9.0
# 再次检查是否有误
lxc profile show cuda
```

然后对于需要使用 CUDA 中的 `nvcc` 的用户的容器，添加这个配置文件：

```shell
lxc profile add tom-container cuda
```

然后由用户自己决定使用哪个版本的 CUDA，在容器内创建软链接 `/usr/local/cuda`，使其指向所需版本即可．例如，使用 CUDA 10.0:

```shell
ln -s /usr/local/cuda-10.0 /usr/local/cuda
```

### 远程管理 LXD

LXD 允许用户通过远程方式管理 LXD 容器．本地端和远程端均需安装 LXD．将本地端的证书 `~/.config/lxc/client.crt` 上传到服务器，然后使用将其添加：

```shell
lxc config trust add client.crt
```

对于每一台使用 LXD 的服务器均需要添加本地端的证书才可以使用．然后，在本地端，我们还需要将服务端的 IP 给加入，不然本地端也不知道去与哪个服务端进行交互嘛．

```shell
lxc remote add remote-name IP
# 列出 remote 端看看
lxc remote list
```

这样子，就可以在本地端直接管理远程的 LXD 容器了，只需要在加入 remote-name 前缀即可．例如，使用 server1 上的镜像 template 来在 server2 上创建一个容器 demo，可以直接使用命令：

```shell
lxc launch server1:template server2:demo
```

LXD 自动地将 server1 上的镜像传输到 server2 上，然后创建一个容器 demo．因为是在局域网内，网络传输速度可达 100M/s，也是非常快速的．

LXD 提供了很多的命令，详情请根据需要查看 `lxc` 的帮助．特别地，每一个命令都会提示其用法，例如 `lxc launch`：

```text
Description:
  Create and start containers from images

Usage:
  lxc launch [<remote>:]<image> [<remote>:][<name>] [flags]
```

我们只需要将 remote 替换为我们的远程服务端名称即可通过远程管理 LXD．若省略 `[<remote>:]` 则默认使用 `local:`，即管理本机上的 LXD．因为 `:` 被用作分隔符，建议不要在容器、镜像名称等中使用该符号，以免解析出错．

### 利用 pylxd 查询容器信息

考虑到如果有多台 GPU 设备，如果每个用户忘记了自己容器的 IP 都需要问管理员也太麻烦了．不才利用 LXD 提供的 python 包 [lxpyd](https://pylxd.readthedocs.io/en/latest/index.html) 结合 [cherrypy](https://cherrypy.org/) 写了一个简单的网页，用于查询各个宿主机上运行的 LXD 容器的信息．具体的代码托管在 [GitHub](https://github.com/hubutui/pylxd-webage) 上，并且可以打包为 Docker 镜像，该镜像已经上传到 [Docker Hub](https://hub.docker.com/r/butui/pylxd-webpage)．这个 Docker 镜像只提供了查询 LXD 容器的状态信息特别是 IP 地址的功能．实际上 pylxd 是提供了完整的管理功能的，有闲情雅致的管理员也可以在此基础上开发，方便自己管理容器．当然，目前 GitHub 上有一个轮子 [lxdui](https://github.com/AdaptiveScale/lxdui/) 提供了类似的功能，但是他并不支持管理远程管理，而且还要求本地安装好 LXD，限制蛮多的．如果本机就是 Linux 的话，安装一个 LXD 也很容易，然后用命令管理也不算太复杂的．

## 其他参考

* https://github.com/shenuiuin/LXD_GPU_SERVER
* https://zhuanlan.zhihu.com/p/25710517
* https://feelncut.com/2018/05/03/145.html
* https://shenxiaohai.me/2018/12/03/gpu-server-lab/
* https://newdee.cf/posts/325daa6a/
* https://blog.yangl1996.com/post/gpu-passthrough-for-lxc/

这些文章都有一定的参考价值，但是都存在不同的缺点．

1. 网络设置使用网桥 NAT，用户需要经过宿主机访问，或者管理员要设置端口转发，比较麻烦．本文采用 macvlan，直接给每个容器分配一个与宿主机同网段的 IP，用户可以直接访问，不需要做端口转发．
2. 需要在宿主机和容器内安装相同版本的驱动，维护比较麻烦．本文直接使用 NVIDIA 提供的 nvidia-container-runtime，宿主机只需要安装好驱动即可，容器内不需要安装驱动．将来有需要的时候，也可以升级宿主机的驱动．
3. 天下文章一大抄，都说用 lxdui 来管理，但是实际上 lxdui 暂不支持远程管理，还不如命令好好用．
4. 我们还特地提供了一个 pylxd-webpage，供有需要的管理员和用户使用．Docker 镜像都已经上传到 Docker Hub 了，有需要的可以直接部署使用．虽然只是提供了 LXD 容器状态查询的功能，但是也是提供了一个可以借鉴的使用 pylxd 的思路．欢迎在此基础上进行开发．
5. 共享目录这部分很多教程都语焉不详，本文特地讲得很啰嗦详细了．这个主要还是方便管理员的管理，跳过自然也是无妨的．
6. 有人提到过不同版本的 CUDA 对编译器的不同要求么？有人好心的提供了多版本共存的 gcc/g++，并且提供了友好简单的设置么？本文独创．

