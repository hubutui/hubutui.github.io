---
title: "调整 LXD 存储池大小的正确姿势"
date: 2019-10-04T00:00:00+08:00
draft: false
---

## 前言

由于历史遗留问题，当初安装系统的时候选择的文件系统不是 btrfs，后来开始用 LXD 的时候，创建的存储池（storage pool）选择了 btrfs loop 设备，且默认的存储池容量偏小．如果存储池所在分区为 btrfs 的话，那就可以直接创建子卷来使用，存储池容量直接与存储池所在分区的大小共享．简单的自然是直接根分区就用的 btrfs，然后直接创建默认的存储池就可以直接共用根分区的了．如果有多余的空闲的硬盘，也可以将该硬盘直接用作 LXD 的存储池，文件系统直接用 btrfs 就很合适了．使用独立的硬盘来作为 LXD 的存储池也是 LXD 官方文档里所建议的，可以用于生产环境的．不过说这些都来不及了，不想重装系统，那就只好想办法调整存储池的大小了．

## How

其实很简单，首先列出所有的存储池：

```shell
lxc storage list
+---------+-------------+--------+--------------------------------+---------+
|  NAME   | DESCRIPTION | DRIVER |             SOURCE             | USED BY |
+---------+-------------+--------+--------------------------------+---------+
| default |             | btrfs  | /var/lib/lxd/disks/default.img | 8       |
+---------+-------------+--------+--------------------------------+---------+
```

可以看到有一个默认的存储池 `default`．我们看看他的信息：

```shell
lxc storage show default

config:
  size: 128GB
  source: /var/lib/lxd/disks/default.img
description: ""
name: default
driver: btrfs
used_by:
- /1.0/containers/demo-container
- /1.0/images/86e50c7d02c9fca2e620e0ac41117e9b99c35779c3595c22c624c6f0b9b8f3e1
- /1.0/images/9d77238fd2ec6a416329c1a86d257fbf1c47d1fafbc7a144b14aa3300432630c
- /1.0/profiles/default
status: Created
locations:
- none
```

可以看到我们创建该存储池的时候指定其大小为 128G．还有另外一个命令查看：

```shell
lxc storage info default

info:
  description: ""
  driver: btrfs
  name: default
  space used: 39.58GB
  total space: 128.00GB
used by:
  containers:
  - demo-container
  images:
  - 86e50c7d02c9fca2e620e0ac41117e9b99c35779c3595c22c624c6f0b9b8f3e1
  - 9d77238fd2ec6a416329c1a86d257fbf1c47d1fafbc7a144b14aa3300432630c
  profiles:
  - default
```

这两个命令的区别在于 `show` 命令显示的 `size` 是创建时候的容量，无法修改的；`info` 命令则显示其当前的状态．

我们给这个存储池增加 128G 的容量：

```shell
truncate -s +24G /var/lib/lxd/disks/default.img
reboot
btrfs filesystem resize max /var/lib/lxd/storage-pools/default
reboot
```

如果你调整的是其他存储池的容量，注意修改命令中的对应路径．最后，我们检查一下其容量：

```shell
lxc storage info default
info:
  description: ""
  driver: btrfs
  name: default
  space used: 39.58GB
  total space: 255.52GB
used by:
  containers:
  - demo-container
  images:
  - 86e50c7d02c9fca2e620e0ac41117e9b99c35779c3595c22c624c6f0b9b8f3e1
  - 9d77238fd2ec6a416329c1a86d257fbf1c47d1fafbc7a144b14aa3300432630c
  profiles:
  - default
```

## 参考

* [Expand lxd’s default btrfs storage pool](https://michael.yoo.id.au/2018/07/29/expand-lxds-default-btrfs-storage-pool/)
* [LXD official Docs](https://lxd.readthedocs.io/en/latest/storage/)