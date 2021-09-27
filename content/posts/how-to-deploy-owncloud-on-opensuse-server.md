---
title: "如何在 openSUSE 服务器上架设 ownCloud 私有云"
date: 2014-10-09T00:00:00+08:00
---

[ownCloud](http://owncloud.org/) 是一个自由且开源的私有云存储解决方案。 用户可以方便快捷的将其部署到服务器上，并拥有完全的控制与修改权利，不用把自己的敏感资料放到 OneDrive, Dropbox, Google Drive 上去了。当然也不会有空间容量限制，只要您的服务器硬盘容量充足。 

## 安装与配置

So，要如何安装部署 ownCloud？ownCloud 社区在 [OBS](http://software.opensuse.org/download.html?project=isv:ownCloud:community&package=owncloud) 上为各发行版打包了软件，包括 CentOS，Debian，Fedora，openSUSE，RHEL 等。打开 ownCloud 的 OBS [项目页面](http://software.opensuse.org/download.html?project=isv:ownCloud:community&package=owncloud)，选择您的系统，即可下载安装包。openSUSE 用户可以直接点击页面上的一键安装按钮即可安装。 

### 配置 ownCloud

安装完成之后还需要一些简单的配置才能使用 ownCloud，进入 `/srv/www/htdocs/` 目录，检查一下 `owncloud` 目录的权限，拥有者为 root，群组为 www，一般是不会有错的。如果你同时也安装了 WordPress，也启用了固定链接，那就还需要编辑文件

```text
/srv/www/htdocs/owncloud/.htaccess
```

找到

```text
<IfModule mod_rewrite.c>
```

这个部分，紧接着

```text
<IfModule mod_rewrite.c>
```

这行，加上这么一行：

```text
Options +FollowSymLinks
```

不知道为什么安装 ownCloud 的时候，`apache2.service` 被关掉了，需要重新启动：

```bash
systemctl start apache2.service
```

### 配置 MySQL/MariaDB 数据库

ownCloud 需要数据库支持，这里使用 MariaDB。其他数据库的操作也是类似的，新建一个用户以及一个数据库给 ownCloud 用就可以了。 可以使用命令行来操作，或者更简单的，使用 phpMyAdmin。直接打开 phpMyAdmin，点击「用户」->「添加用户」，然后填写用户名，密码，假设用户是 owncloud；勾选「创建与用户同名的数据库并授予所有权限」，最后点执行。ownCloud 所需的数据就建好了。 

## 防火墙设置

这个没什么好说，直接打开 YasT->「防火墙」->「允许的服务」，添加 HTTP Server 和 HTTPS Server 即可。 

## 差不多最后一步了

浏览器打开 [www.example.org/owncloud](http://www.example.org/owncloud)，这里 www.example.org 换成自己的域名。按照提示设置您的 ownCloud 用户名与密码，写上数据库名和数据库用户以及密码，确认下一步就安装了。 安装好之后就进入进入 ownCloud 欢迎页面了，您可以下载安装客户端了。 

## ownCloud 客户端

ownCloud 提供了 Windows, Mac OSX, 多个 Linux 发行版的客户端，同时也提供了 iOS 和 Android 客户端。Android 客户端需要付费 6.06 元人民币(PS: Google Play 商店也收人民币了？)。 

## 参考

  * openSUSE wiki: [SDB: ownCloud](https://en.opensuse.org/SDB:OwnCloud)
  * Blog from HowtoForge: [How To Install ownCloud 7 Server and Client on OpenSuse 13.1](http://www.howtoforge.com/how-to-install-owncloud_7-server-and-client-on-opensuse-13.1)
  * ArchLinux wiki: [ownCloud](https://wiki.archlinux.org/index.php/OwnCloud)
