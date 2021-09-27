---
title: "更新 CentOS 7 内核以正常使用 Intel I219-LM 网卡"
date: 2021-09-27T08:00:00+08:00
---

## 起因

有一台新机器，型号是 [HP ProDesk 480 G7 PCI](https://support.hp.com/cn-zh/document/c06938386)，需要安装 CentOS 7 系统．但是安装过程中发现没看到网卡，先不管，装完再说．经过查询，网卡是 Intel Ethernet I219-LM，驱动在较新的内核（据[此文](https://linux-hardware.org/index.php?id=pci:8086-15bb-103c-83e0)，4.12 版本的内核开始支持）里就有了的．但是 CentOS 7 的内核版本是 3.10．

## 解决方法

既然如此，解决方法就很简单了．只需要更新一下内核即可．我们可以使用 [ELRepo](https://elrepo.org/tiki/HomePage) 提供的内核．不过，我们目前这台机器无法连接网络，因此必须将需要安装的包预先下载到本地，然后复制到这台电脑上去安装．

首先，使用一台有网络的电脑，找一个 CentOS 7 的环境．这里，我们使用 Docker：

```bash
docker run -it --rm -v $PWD/rpms:/data centos:7 bash
```

进入 Docker 容器之后：

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install -y https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install -y --downloadonly --downloaddir /data kernel-lt
exit
```

执行完上述命令之后，我们就将 `kernel-lt` 及其依赖包下载好了，其中 `lt` 表示 `long-term`，你也可以安装 `kernel-ml`，即 `mainline` 版本的内核．就我们当前的问题而言，`kernel-lt` 就足够了．

然后，我们还需要下载一个 GPG 公钥：

```bash
wget https://www.elrepo.org/RPM-GPG-KEY-elrepo.org -O rpms/key.pub
```

现在，将 `rpms` 文件夹复制到要更新内核的机器上．使用以下命令安装：

```bash
rpm --import rpms/key.pub
yum localinstall -y rpms/*.rpm
```

安装完毕之后重启系统，开机时按 `ESC` 键，选择进入 BIOS 设置页面，将 Security Boot 给关闭掉，保存并重启．此时系统会提示你输入一串数字确认，按照提示输入即可．然后正常启动，进入 GRUB2 菜单时，注意选择启动进入新版本的那项．

进入系统之后，检查一下网络是否正常．如无意外，应该是没有问题了．此时，可以把旧版本的内核给删掉了：

```bash
yum remove kernel
```

## 参考

- [ELRepo Project](https://elrepo.org/tiki/HomePage)
