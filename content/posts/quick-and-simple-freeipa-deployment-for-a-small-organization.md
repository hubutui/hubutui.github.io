---
title: "适用于小型机构（例如实验室）的 freeipa 快速部署指南：服务端与客户端配置详解"
date: 2024-12-03T00:00:00+08:00
toc: false
images:
tags:
  - freeipa
  - 统一身份认证
  - ipa
  - idm
---

本文介绍如何快速简单部署 freeipa 服务和配置客户端，以满足小型机构（比如一个实验室的集群）的局域网使用．

## 基础配置

以下是我们准备的基本配置：

1. freeipa 域名：`ipa.example.local`，一般这个域名不用实际存在的域名，因为我们是局域网配置．
2. 一台 Rocky Linux 9.5 的服务器作为 freeipa 的服务器，IP 地址为 `192.168.1.11`，主机名为 `ipaserver.ipa.example.local`．需要注意的是，安装 freeipa 服务端的服务器，主机名必需是 freeipa 域名 `ipa.example.local` 下的，且必需是完全限定域名（FQDN），比如我们的例子 `ipaserver.ipa.example.local` 就是满足要求的．其他的比如 `ipaserver` 或者 `ipaserver.ipa.baidu.local` 是不满足的．
3. 一台 Rocky Linux 9.5 或者 Ubuntu 22.04 的机器，作为需要加入 freeipa 域的设备之一，IP 地址为 `192.168.1.12`，主机名为 `client01.ipa.example.local`．这里并不要求其主机名必需在 `ipa.example.local` 下，但是必需是 FQDN．特别补充，如果你是用虚拟机来进行测试，并且克隆了一个虚拟机，那他们的指纹是相同的，克隆后的虚拟机加入 freeipa 的时候会提示他已经加入．此时，应该删除 `/etc/ssh/ssh_host_*` 文件，然后执行命令 `ssh-keygen -A`，同时还建议修改 hostname 为不同的名字，再次加入即可．

## 服务端

具体步骤：

1. 安装必需的包：`dnf install freeipa-server`
2. 执行安装：

```bash
ipa-server-install --realm IPA.EXAMPLE.LOCAL --domain ipa.example.local --ds-password poiuytrewq --admin-password poiuytrewq --mkhomedir --hostname ipaserver.ipa.example.local --skip-mem-check --unattended
```

解释：

1. `--ds-password` 和 `--admin-password` 分别指定 Directory Manager 和 ipa 管理员密码．我们一般只需要用到 ipa 管理员，前者几乎不用的．密码要牢记．
2. `--no-hbac-allow` 生产环境一般要用这个，不要添加默认的 `allow_all` 的 HBAC 规则．这个规则的意思是允许所有用户直接登录加入 freeipa 域的设备．这里我们的使用情况比较简单，自己添加规则还比较麻烦，这里不做演示．
3. `--realm` 指定域，这个一般就是域名的大写．

添加一个用户，我们可以使用命令行或者网页操作，使用命令行操作如下：

```bash
# 先获取 admin 用户的凭证，然后才能使用 ipa 命令
kinit admin
# 添加用户 jim，这个是建议用的最少的选项
# 不指定 shell 则默认为 /bin/sh，一般不够好用
# --cn 指定全名
# --random 会生成随机的密码
# 最后的参数 jim 为登录用户名
# 这里可以设置 --sshpubkey 为用户提供的 ssh 公钥，这样用户就要可以直接用 ssh 登录，不需要设置密码
ipa user-add --random --shell /bin/bash --first Jim --last Hacker --cn JimHacker --sshpubkey "ssh-rsa AAAAB3NzaC1yc2EAAAAD xxx" jim
```

执行完毕，命令会输出这个创建好的用户的详细信息，包括生成的随机密码，用户可以使用这个密码进行登录，首次登录就会被要求修改密码．

使用网页的话，需要编辑你需要使用网页管理的机器的 `/etc/hosts` 文件，使得这个机器能够解析 freeipa 服务器 `ipaserver.ipa.example.local` 解析到对应的 IP 即可访问．例如：

```text
192.168.1.11 ipaserver.ida.example.local
```

然后浏览器访问 https://ipaserver.ipa.example.local，浏览器会提示你不安全，因为这个证书是我们自己签发的，点击信任即可继续．使用浏览器操作会比较简单直观，也可以添加一下用户的 SSH 公钥，用户登录就不需要密码了．

## 客户端

安装必需的包，Rocky Linux 使用命令：`dnf install -y ipa-client`，Ubuntu 使用命令 `apt install -y freeipa-client`

将设备添加到 freeipa 的域里来管理，需要配置这个设备的 DNS 解析，让他能够解析 ipaserver．还是最简单的方式，直接修改 `/etc/hosts`，内容同前．

同样的，客户端的 hostname 也要是 FQDN，但是不要求一定是 `ipa.example.org` 下的，比如你的 hostname 为 `pcpu01.ipa.google.local`，也不影响加入．

这个操作需要有被添加到 freeipa 域的设备的管理员，以及 freeipa 的 admin 用户的权限，执行命令：

```bash
sudo ipa-client-install --unattended --mkhomedir --principal admin --password poiuytrewq --domain ipa.example.local --server ipaserver.ipa.example.local
```

这里 `--mkhomedir` 选项可以在用户登录该设备的时候自动创建其 home 目录．

然后就可以从任意一台可以访问到这台设备的电脑上，通过 ssh 登录到这个机器上了．如果管理员在添加用户的时候，设置了 ssh 公钥，则此时使用对应的私钥即可免密登录，否则应该用密码登录．例如，jim 用户登录可以使用命令 `ssh jim@192.168.1.12`，未设置 ssh 公钥则会提示输入密码．

普通用户配置自己电脑的 DNS 解析后，也可以通过网页 https://ipaserver.ipa.example.local 登录自己的帐号，在上面修改自己的信息．
