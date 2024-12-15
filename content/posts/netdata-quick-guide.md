---
title: "简单易用的netdata安装与配置指南"
date: 2024-12-15T00:00:00+08:00
toc: false
images:
tags:
  - netdata
  - resend
  - msmtp
---

## 前言

我需要在服务器上配置一个简单已用的系统状态监控服务，并且希望这个它能够在系统异常的时候，如 CPU 使用率异常过高，存储不足等情况，及时发送邮件给我提示。

这种一般可以考虑 netdata, open-falcon, prometheus, kafka 等等，综合考虑下来，netdata 是安装配置都最简单的，基本上开箱即用。

## 安装

虽然 Ubuntu, Debian, Rocky Linux 等都有打包提供了 netdata，但是版本一般相对比较老旧，建议从 netdata 官方安装，他们也有自己的仓库提供 deb 或者 rpm 安装．使用以下命令安装即可：

```bash
wget -O /tmp/netdata-kickstart.sh https://get.netdata.cloud/kickstart.sh && sh /tmp/netdata-kickstart.sh --no-updates --stable-channel --disable-telemetry
```

实际上就是从 netdata 那边下载了一个脚本，这个脚本会根据你的系统自动配置好对应的软件源，然后从软件源里安装．详细可以参考官方的[安装文档](https://learn.netdata.cloud/docs/netdata-agent/installation/linux)．

然后是启动 netdata 服务，并设置开机启动：

```bash
systemctl start netdata
systemctl enable netdata
```

启动之后我们就可以通过 http://IP:19999 查看 netdata 的监控页面了。点击页面右下角的 `Skip and use the dashboard anonymously` 即可查看面板，不需要注册和登录，那个是 netdata cloud 的功能，需要另外的配置，不在本文范围内．其中 IP 为安装了 netdata 的服务器的 IP．默认 netdata 绑定 IP 设置为 `*`，如果你不想让本机之外的其他机器访问，可以设置 netdata 的绑定 IP 为 `127.0.0.1`，使用命令 `edit-config netdata.conf` 打开 netdata 的配置文件，找到 `[web]` 下的 `bind to`，改成 `127.0.0.1`。重启 netdata 服务之后即可生效。

如果 `edit-config netdata.conf` 打开看到的 netdata 配置文件没有内容，你可以按照提示使用命令生成当前的配置，即 `netdatacli dumpconfig > /etc/netdata/netdata.conf`．

注意 `edit-config` 这个命令是 netdata 这个包里的，官方的版本是放在 `/etc/netdata/edit-config`，不同的发行版自己打包的可能路径不同，且一般都不在 PATH 里，需要根据情况写完整的路径，后文的 `alarm-email.sh` 也是．CentOS/Rocky Linux 可以使用 `rpm -ql netdata` 查看所需命令对应文件的完整路径，Ubuntu 可以使用 `dpkg -L netdata` 查看．

## 配置邮件告警

配置邮件告警一般需要的是一个发送邮件的程序，常用的有 mutt, msmtp 等，这里我们使用最简单的 msmtp。

1. 安装 msmtp，`apt install msmtp` 或者 `yum install msmtp` 即可。
2. 编辑 msmtp 的配置文件 `/etc/msmtprc`，内容如下：

```text
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-bundle.crt
logfile        /var/log/msmtp.log

account        default
host           smtp.your-domain.com
port           465
tls_starttls   off
from           no-reply@your.domain
user           your-user-name
password       your-password
```

其中的 smtp 服务器和账户密码配置根据你的邮件服务商的设置来，或者如果你有自己的域名，推荐使用 [resend](https://resend.com)。`tls_trust_file` 的设置不同的系统可能具体文件路径不同，根据实际情况设置即可。例如这里给出的 `/etc/ssl/certs/ca-bundle.crt` 是 CentOS 的，Ubuntu 的是在 `/etc/ssl/certs/ca-certificates.crt`。

然后是使用命令 `edit-config health_alarm_notify.conf` 编辑 netdata 配置文件 `health_alarm_notify.conf`：

1. `sendmail` 设置为 `/usr/bin/msmtp`。
2. 设置 `DEFAULT_RECIPIENT_EMAIL="username@your-domain.com"`，即指定接收邮件的邮箱地址，这里可以设置多个邮件地址，以空格隔开即可。
3. `EMAIL_SENDER` 可以不用设置，因为前面 `/etc/msmtprc` 已经设置了了 `from`．

设置完毕之后可以使用 `/usr/libexec/netdata/plugins.d/alarm-email.sh test` 尝试发送测试邮件，检查看邮件是否能够发送成功．

全部配置完毕，重启一下 netdata 服务即可，`systemctl restart netdata`．在配置邮件通知前就已经有的告警可能不会重新发送邮件提醒，如果有需要应该自行处理这些告警信息．

## 参考

1. netdat 官方[安装文档](https://learn.netdata.cloud/docs/netdata-agent/installation/linux)．
2. netdata 官方文档关于[邮件通知](https://learn.netdata.cloud/docs/alerts-&-notifications/notifications/agent-dispatched-notifications/email) 设置的部分．
