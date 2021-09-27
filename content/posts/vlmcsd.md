---
title: "vlmcsd service in Linux"
date: 2018-05-11T00:00:00+08:00
---

vlmcsd 是一个 KMS 服务端软件，用于 M$ 产品的激活与授权．

## 安装

自己去[官网](https://github.com/Wind4/vlmcsd)下载源码编译打包．Arch Linux 用户可以去 [AUR](https://aur.archlinux.org) 找现成的 PKGBUILD 脚本打包．

## 启动服务

Arch Linux 的 AUR 中的 vlmcsd 包有提供一个 systemd unit，内容如下：

```text
[Unit]
Description=KMS Emulator

[Service]
Type=forking
User=nobody
ExecStart=/usr/bin/vlmcsd

[Install]
WantedBy=multi-user.target
```

可以根据上述内容自己创建一个，然后复制到合适的路径．启动并设置开机启动服务：

```shell
systemctl start vlmcsd
systemctl enable vlmcsd
```

vlmcsd 的默认端口为 1688，如果有防火墙的请修改防火墙设置．

## 使用方法
### Windows

打开 cmd，使用命令安装批量授权 key：

```shell
slmgr /ipk xxxxx-xxxxx-xxxxx-xxxxx
```

其中批量授权 key 可以从[这里](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/jj612867(v=ws.11))根据 Windows 版本选择对应的 key．

设置 KMS 服务器地址：

```shell
slmgr /skms IP
```

其中 IP 为你的 vlmcsd 服务的 IP．然后激活：

```shell
slmgr /ato
```

### M$ Office

切换你的 M$ Office 安装目录，设置 KMS 服务器地址：

```shell
cscript ospp.vbs /sethst:IP
```

激活：

```shell
cscript ospp.vbs /act
```

## 可能需要注意的问题

支持的 Windows 版本参考[这里](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/jj612867(v=ws.11))，M$ Office 仅支持批量授权版的．
