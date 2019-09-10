---
title: "联想新路由 newifi mini 刷入 OpenWRT"
date: 2015-10-16T00:00:00+08:00
---

PandoraBox 其实是基于 OpenWRT 二次开发的, 其主页上根本就没有提供源码的嘛, 为了支持开源事业, 为了安装简洁的原版的~~**纯净的钙**~~ OpenWRT, 为了**爱与和平**, 我决定自行刷入官方的 OpenWRT 固件. 

## 升级 uboot

如果 newifi mini 的 uboot 还没升级, 那么刷入固件的时候就会出错, 无法刷入. 所以第一步就是升级 uboot. 这部分的内容可以去 newifi 的官方论坛查找, 此处省略. 

## 刷入固件之前

OpenWRT 固件的大小仅为 3M 左右, 非常非常的节省空间. 即使加上后续需要安装的软件包, 也会比直接刷入 PandoraBox 这个庞然大物要节省了不少空间. 但是, 必须要注意的是, 由于这个无线路由器使用的是联发科的芯片, 并且没有开源驱动提供, 5G 基本是不能用的. 2.4G 基本是够用的了, 不然就别往下看了. 

### 下载固件

直奔官网下载, [openwrt-15.05-ramips-mt7620-Lenovo-y1-squashfs-sysupgrade.bin](http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/openwrt-15.05-ramips-mt7620-Lenovo-y1-squashfs-sysupgrade.bin); 如果速度太慢可以使用中科大的镜像. 一般来说使用最新的版本 15.05 就可以了, 不建议使用 snapshot 版本. 下载完成后必须校验哈希值, 否则搞错了找谁哭去啊. 

### 通过不死 uboot 刷入

这是刷机的基本技能, 基本上 uboot 还在就可以通过刷入正确的固件使 newifi 正常工作, 不用怕的. 拔掉 WAN 口网线, 拔掉电源, 按住复位键, 插入电源, 等待数秒钟, 电源灯和 USB 灯同时闪烁即可松手, 顺利进入 uboot 模式. LAN 口网线另一头连接电脑, 设置电脑连接方式为静态地址分配, 给电脑随意设置一个 IP (192.168.1.2-192.168.1.255), 网页浏览器打开 192.168.1.1, 选择固件, 刷入, 耐心等待 100 秒, 升级成功. 

## 刷入固件之后

网线接回 WAN 口, 重启电源, 电脑连接方式改成 DHCP 方式. 网页打开 192.168.1.1, 设置密码, 登录. 如果其他设备已经使用了 192.168.1.1 这个 IP, 可能会有冲突, 导致无线路由器无法上网, 建议修改 LAN 口 IP 为 192.168.x.1, 其中 x 为 1~255 任一整数, 保存后重启路由器, 电脑重联网. 这回网页打开的地址就是 192.168.x.1 了. 

### 修改 opkg 软件源

国内有中科大提供的镜像, 速度不错, 可以考虑自行修改 `/etc/opkg.conf`, 然后还可以加入以下几行: 

```text
arch all 100
arch noarch 200
arch ralink 300
arch ramips_24kec 400
```

最后记得刷新软件源: 

```bash
opkg update
```

### 中文支持

默认 OpenWRT 的网页界面 luci 只有英文, 需要中文的可以自行安装相关语言包. ssh 登录后: 

```bash
opkg install luci-i18n-base-zh-cn luci-i18n-firewall-zh-cn
```

然后可以根据需要安装常用软件包: 

```bash
opkg install wget aria2 amule screen luci-app-samba luci-i18n-samba-zh-cn
```

OpenWRT 15.05 没有提供 shadowsocks-libev, 但是 snapshot 版本却有, 可以考虑直接拿过来用. USB 支持: 

```bash
opkg install kmod-usb-storage
```

需要其他文件系统支持的可以看看 `kmod-fs` 开头的软件包: 

```bash
opkg list | grep kmod-fs
```

你可能需要 `fdisk`: 

```bash
opkg install fdisk
```

OpenWRT 竟然还提供了 `cfdisk`, 如果有需要的话可以安装. 还有 `lsusb` 命令, 由 `usbutils` 提供, 个人觉得这个没必要的.其实, `ntpclient` 也没有默认安装的, 也可以装一个, 还是很有用的. 

### 多写一点吧, aria2c 下载

aria2c 支持 rpc, 所以可以参考[这篇文章](http://www.ytyzx.net/index.php?title=%E8%B7%AF%E7%94%B1%E5%99%A8OpenWrt%E5%A6%82%E4%BD%95%E8%84%B1%E6%9C%BA\(%E7%A6%BB%E7%BA%BF\)%E4%B8%8B%E8%BD%BDBT%E6%96%87%E4%BB%B6)写个配置文件, 然后运行 screen, 启动 aria2c, 按 CTRL+SHIFT+A+D 断开 screen 会话, 但是 aria2c 还在后台跑; 恢复会话的时候使用 `screen -r`; 还可以按 CTRL+C 停止 aria2c 会话并且保存起来, 下次启动 aria2c 的时候继续下载. 

### 无线设置

直接打开 luci -> 网络 -> 无线, 就可以自行设置了, 很简单的啦. 如果你真的想用 5G, 安装 `kmod-mt76` 试试看, 或者去 downloads.openwrt.org.cn 找找 `kmod-mt762x-mmc` 和 `kmod-mt76x2e` 试试看吧. 就这么多吧.

## 2017 年更新

现在可以考虑刷入 [LEDE](https://lede-project.org), 方法与上述无异. 并且 LEDE 添加了 newifi 的 5G 驱动, 开箱即用. 另外需要注意的是 LEDE 的软件源设置与 OpenWRT 略有不同, 详情可参考 LEDE 的[文档](https://lede-project.org/docs/user-guide/opkg#adjust_repositories). 此外, 中科大提供有 LEDE 的[镜像](http://mirrors.ustc.edu.cn/lede/).
