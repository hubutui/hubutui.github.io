---
title: "利用 cURL 进行 DR.COM 的网页认证"
date: 2017-08-12T00:00:00+08:00
---

## 前提

DR.COM 允许使用网页认证，也就是直接连接网络之后，浏览器打开任意网站可以跳转到指定的地址，然后输入帐号和密码进行认证．

## 基本原理（我瞎猜的

据我胡说，这应该就是给指定服务器发送一个 HTTP POST 请求，然后就可以上网了．
所以理论上只要我们能够模拟一下这个登录页面，也向服务器发送一个 HTTP POST 请求，
即可顺利上网．

## 抓取 HTTP POST 请求

那首先就需要抓取 HTTP POST 请求，看看认证过程中需要发送什么内容给认证服务器咯．
Chrome 浏览器打开任意一个网站，跳转到认证页面之后，右键->检查，打开开发者工具，
选择 network，勾选 Preserve log．在登录页面填写帐号信息，点击登录，
即可看到相关的 HTTP 请求，找到 Request Method 为 POST 的那个，右键->Copy->
Copy as cURL，即可得到认证所需的 curl 命令．使用该命令即可进行登录认证，
无需在打开网页之后跳转到认证页面进行网页认证了．
而 curl 支持多个平台的，理论上这方法是非常通用的．

## OpenWrt/LEDE 路由器的认证

如果你的路由器刷了 OpenWrt/LEDE，那就可以安装 curl 然后利用 curl 来进行认证．
不过 LEDE 的 curl 的默认编译选项没有添加 zlib 支持，
也就不能使用 `--compressed` 选项，但是我们这里会用到．所以最好自己用 LEDE-SDK 
自行编译一个 `libcurl`，添加上 zlib 支持即可．

假设将之前保存的 curl 命令写入到脚本中 `drcom.sh`，记得添加一行到脚本最前面：

```bash
#!/bin/sh
```

给 `drcom.sh` 添加可执行权限：

```bash
chmod +x drcom.sh
```

直接直接运行即可成功认证：

```bash
./drcom.sh
```

测试认证成功之后，你可以用重定向将 curl 命令的返回，也就是 HTTP POST 的 Response，重定向到 `/dev/null`，忽略输出，即在 curl 命令的那行末尾加上
`&> /dev/null`．

最后建议将该脚本加入到路由器的开机启动脚本即可．

## 参考

* [python 实现的 DR.COM webauth](https://github.com/agentmario/drcom-webauth)

