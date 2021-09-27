---
title: "CentOS 中删除 swap 分区的正确姿势"
date: 2021-05-10T00:00:00+08:00
---



CentOS默认使用 LVM．假设我们要删除的 swap 分区为 `centos/swap`，其中 `centos` 为卷组名称，`swap` 为逻辑卷名称，则正确删除 swap 分区，并把空闲空间分配给 `root` 的步骤如下：

1. 禁用 swap，`swapoff /dev/centos/swap`．如果有程序使用了 swap，这里需要耐心等待系统将 swap 中的内容删除或者转移到内存上．
2. 删除 swap 分区，`lvremove centos/swap`，按照提示选择确认．
3. 将空闲空间分配给 root 分区，你也可以根据需要分给其他的逻辑卷，`lvresize -l +100%FREE --resizefs centos/root`
4. 修改 `/etc/fstab`，删除涉及 swap 的行．这是很显然的，否则启动的时候会挂载出错．
5. 编辑 `/etc/default/grub`，删掉 swap 那部分．这里需要特别注意，CentOS 中在 `GRUB_CMDLINE_LINUX` 设置了 swap 分区，虽然实际上根据 ArchLinux 的[文档](https://wiki.archlinux.org/title/GRUB#LVM)，我们只需将 `lvm` 添加到 `GRUB_PRELOAD_MODULES`列表中即可．而且 RHEL7 [文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/s1-swap-removing)未有提及，容易出错．若不做相应的修改，将导致系统启动时报错找不到 swap 分区．
6. 生成新的 grub 配置文件，`grub2-mkconfig -o /etc/grub2.cfg`．如果用的 UEFI 的话，grub 配置文件路径有所不同，`grub2-mkconfig -o /etc/grub2-efi.cfg`
7. 至此，swap 分区已经正确删除，并将空闲空间分配给了 root 分区，用户已经可以使用了．有必要的话可以重启确认系统能够正确启动．
