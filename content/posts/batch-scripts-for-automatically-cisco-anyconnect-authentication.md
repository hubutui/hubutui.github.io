---
title: "使用批处理脚本进行Cisco Anyconnect快速免输密码登录认证"
date: 2025-02-26T00:00:00+08:00
draft: false
toc: false
images:
tags: 
  - cisco anyconnect
  - VPN
  - 批处理
---

## 前言

因为工作需要，我们需要使用 Cisco Anyconnect 登录 VPN，但是这个客户端做得不太好，不能够记录密码，每次登录都需要手动输入密码，非常麻烦．经过一番搜索与研究，发现他其实提供有一个命令行，方便用于登录．

## 具体做法

实际上就是执行 Cisco Anyconnect 安装目录下的 `vpncli.exe` 命令，他有个交互式的命令行界面，同时也支持输入一个 `profile` 来自动化处理．于是，我们可以写这么一个简单的脚本和配置文件．

脚本文件 `connect-vpn.bat` 内容如下：

```cmd
@echo off
"C:\Program Files (x86)\Cisco\Cisco Secure Client\vpncli.exe" disconnect
echo Starting OpenConnect VPN connection...
"C:\Program Files (x86)\Cisco\Cisco Secure Client\vpncli.exe" -s < "%~dp0credit.txt"
pause
```

这里替换 `vpncli.exe` 为你的实际路径．

然后是 `credit.txt` 文件：

```text
connect vpn.example.org
0
USERNAME
PASSWORD
```

这里实际上就是在执行 `vpncli.exe` 进入交互式界面后依次输入的内容．具体来说：

1. 第一行表示连接到服务器 `vpn.example.org`，这里的服务器根据实际情况修改．
2. 有的服务器有不同的用户组，这时候需要输入一个序号选择．如果没有，这不需要第二行．
3. 第三行输入你的用户名．
4. 第四行输入你的密码．

以上两个文件都保存在同一个目录下．

此外，为了方便，我们还可以写一个批处理脚本来断开 VPN 连接，编辑文件 `disconnect-vpn.bat`，内容如下：

```cmd
@echo off
"C:\Program Files (x86)\Cisco\Cisco Secure Client\vpncli.exe" disconnect
pause
```

使用的时候，直接双击 `connect-vpn.bat` 即可连接到 VPN，想要断开 VPN 只需要双击一下 `disconnect-vpn.bat`．为了方便，我在这两个脚本的末尾都加上了一个 `pause` 命令来暂停，等待用户输入任意键后结束．这样子方便用户查看脚本启动的日志．如果不想要这个，直接删掉改行即可．
