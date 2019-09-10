---
title: "Arch Linux 下通过 CUPS 的网页界面添加网络打印机"
date: 2017-09-30T00:00:00+08:00
---

## 前提

局域网已经有了一台打印机，它的 IP 地址已知，可以直接通过 [http://IP:631](http://IP:631) 这样的网页直接打开管理页面．现在只是需要将其添加到 Arch Linux，这样打印的时候就可以直接使用了．

## 安装 cups

CUPS (以前称作 Common Unix Printing System) 的安装很简单的：

```shell
pacman -S cups
```

然后启动 `cups-browsed.service` 服务：

```shell
systemctl start cups-browsed.service
systemctl enable cups-browsed.service
```

## 添加网络打印机

浏览器打开 [http://localhost:631](http://localhost:631)，进入 CUPS 的网页管理界面．Administration→Add Printer，根据提示输入用户名 root 和对应的密码．然后选择 Other Network Printers→Internet Printing Protocol (http)，输入你的打印机IP地址和端口，点 continue，根据需要设置一下 Name, Description, Location，点 continue，Make 那里选择打印机的厂商和星号，根据个人经验，这点可以随便选，似乎没什么区别，继续点 Add Printer．最后设置一下打印机的参数，一般只需要将纸张类型改为 A4，其他都不需要修改．

其实在 Add Printer 这一步的时候，也可以看到 cups 发现的局域网内的打印机，可以直接添加，但是后面在选择打印机型号的时候会遇到错误，只需要修改打印机型号即可．看起来添加网络打印机并不需要这里设置的型号与实际的型号一致．

## 添加虚拟打印机

此外还可以添加一个虚拟打印机，这样只要你打开的文档可以打印，它就可以直接打印为 PDF 文档，无需特别的转换．要使用虚拟打印机，我们需要安装 `cups-pdf` 包：

```shell
pacman -S cups-pdf
```

这个比起 Windows 下添加的各种虚拟打印机要简单多了吧．同样在 [http://localhost:631](http://localhost:631) 中添加虚拟打印机即可．

## 参考

1. 维基百科 [CUPS](https://zh.wikipedia.org/wiki/CUPS)
1. Arch Linux wiki: [CUPS](https://wiki.archlinux.org/index.php/CUPS)
1. CUPS 的[官方主页](https://www.cups.org)
