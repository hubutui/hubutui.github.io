---
title: "SSH 代理转成 HTTP 代理并共享给其他设备使用"
date: 2014-11-25T00:00:00+08:00
---

妈蛋！我的手机终于连上 Google Play 了。自从升级了 Android 5.0, SSH tunnel 竟然不能用了！真是丧心病狂。本来想搭个 VPN，尼玛根本搞不掂啊。无意之中发现 GoAgent 可以共享给手机，那么说 SSH 代理也可以咯？果然啊！ 

## 电脑上

基本条件是这样，电脑很容易可以使用 SSH 代理，其他设备，比如手机，跟电脑在同一个子网下，譬如在同一个 WiFi。 一般的 SSH 代理需要运行这个命令：

```bash
ssh -qTfnN -D 7070 user@server:port
```

改成这样：

```bash
ssh -qTfnN -D 0.0.0.0:7070 user@server:port
```

user 和 server 以及 port 当然按照自己的服务器来写。0.0.0.0 是个 Magic Number，详情请 Google。 这样还不够，还是把 SSH 代理转成 HTTP 代理更加通用。这里需要一个软件，privoxy。直接从软件源里安装就可以了吧。

```bash
zypper install privoxy
```

写个配置文件，假设是 `~/privoxy-config`，内容为：

```text
forward-socks5 / 127.0.0.1:7070 .
listen-address 0.0.0.0:8080
```

不要漏掉第一行末尾的那个点。启动 privoxy：

```bash
privoxy ~/privoxy-config
```

电脑端这就搞好了。 

## 手机这边

手机上，我只说 Nexus 5，其他不详，打开 WLAN，长按连接的 wifi，打开修改网络这项。然后设置代理，代理服务器主机名写电脑的 IP，最好用路由器给分配个固定的局域网 IP；代理服务器端口写 8080,保存即可。浏览器打开 Google 之类的网页测试一下，或者在线查一下 IP 也行。 Done。
