---
title: "LVM 实战：ArchLinux 上将 home 分区放在两块硬盘组成的 LVM 上"
date: 2018-07-15T12:54:01+08:00
---

# 情况简介

已经安装好的 Arch Linux，分区情况如下：

```text
/dev/sda1	/boot
/dev/sda2	/
/dev/sdb1	/home
```

现在新增一块硬盘 `/dev/sdc`，希望可以将 `/dev/sdc` 分给 `/home` 分区．理所当然地想到使用 LVM．

# 实战

首先是对 `/dev/sdc` 分区，直接使用 gdisk 工具，分一个 Linux LVM 分区，gdisk 中的分区类型为 `8e00`．这一个硬盘只分一个分区 `/dev/sdc1` 即可．

然后创建物理卷（physical volume, pv），

```shell
pvcreate /dev/sdc1
```

创建卷组（volume group, vg），这里设定卷组名称为 homevg，

```shell
vgcreate homevg /dev/sdc1
```

创建逻辑卷（logical volume, lv），这里设定逻辑卷的名称为 homelv，并将卷组的所有空闲空间分给它，

```shell
lvcreate -l 100%FREE homevg -n homelv
```

创建好的逻辑卷在 `/dev/<volume_group>/<logical_volume>` 下，可以将它看作我们平时使用的类似 `/dev/sda1` 分区，将其格式化，并挂载即可使用．

创建文件系统，也就是格式化分区，我们这里使用 XFS 文件系统：

```shell
mkfs.xfs /dev/homevg/homelv
```

将逻辑卷挂载，

```shell
mount /dev/homevg/homelv /mnt
```

将 `/home` 分区的内容复制到逻辑卷，

```shell
rsync -Praz /home /mnt
```

备份完成之后，修改 `/etc/fstab`，使用逻辑卷 `/dev/homevg/homelv` 作为新的 `/home` 分区，重启测试．不会编辑 `/etc/fstab` 的可以使用找个 Arch Linux ISO，进入 Live USB 之后，挂载分区，

```shell
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
mount /dev/homevg/homelv /mnt/home
```

然后生成 `/etc/fstab`，

```shell
gen-fstab -U /mnt > /mnt/etc/fstab
```

经过测试正常使用之后，再将原来的 `/home` 分区，也就是 `/dev/sdb1` 删除，创建物理卷，添加到卷组，

```shell
pvcreate /dev/sdb1
vgextend homevg /dev/sdb1
```

然后扩展逻辑卷，

```shell
lvresize -l +100%FREE --resizefs homevg/homelv
```

# 参考
[Arch Wiki: LVM](https://wiki.archlinux.org/index.php/LVM)
