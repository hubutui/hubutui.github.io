---
title: "xpra 快速入门指南"
date: 2025-11-09T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - xpra
  - 远程桌面
  - remote desktop
---

## 简介

[xpra](https://github.com/Xpra-org/xpra) 可以被看做是 X11 的 screen 或者 tmux。也就是说，用户可以利用 xpra 做到在服务器启动一个应用程序，但是将它转发到本地机器来显示，同时还能断开连接，并随时在任何机器上恢复连接，保持应用程序的工作状态不变。

此外，它还支持直接 Windows 和 MacOS 的桌面共享，不过这并不是本文的重点。这里我们主要关注的是 xpra 如何在 Linux 服务器上启动一个应用程序，然后在本地端（Windows、Linux 系统都可以）显示，从而实现某种程度上的远程桌面连接。

按照 xpra 的设计，应用程序的资源消耗都是在 xpra 服务端的，本地客户端仅作显示，从而可以充分利用服务器的计算能力，并节省本地机器的计算资源。

## 安装

不同的发行版安装方式不同，请直接参考[官方文档](https://github.com/Xpra-org/xpra/wiki/Download)。建议直接使用 xpra 官方直接提供的源，而不是从自己的发行版软件源安装，否则可能由于版本太过老旧而遇到各种新版本已经解决的问题。

ArchLinux 用户可以从 ArchLinux 源里安装，但是目前（2025-11-09）版本是落后于 xpra 官方的，可以直接下载 xpra 的 PKGBUILD 文件后修改版本号直接构建：

```bash
pkgctl repo clone --protocol=https xpra
cd xpra
# 编辑 PKGBUILD 并修改 pkgver 为最新版本
# 然后更新源码包的 checksums
updpkgsums
# 打包
extra-x86_64-build -- -- --skippgpcheck
# 最后用 pacman -U 命令安装即可
```

这样，我们就安装好了服务端。客户端则比较简单，一般我们用 Windows 去连接，直接从 xpra 官网下载安装包并双击执行即可。

## 使用

xpra 的命令比较繁杂，它支持多种连接和启动方式，这里我们只讲最有用的。

我们建议分别在服务端和客户端操作，明确分开，并且连接只使用 SSH 方式，它提供了不错的加密和安全。TCP 方式仅在可靠的网络环境下使用，它存在比较明显的安全隐患，一般不建议使用。xpra 虽然支持 SSL 方式，但是简单起见，这里不做叙述。

同时，这里我们还考虑到中文输入法的支持，优先考虑的是 fcitx5，如需 ibus，也可以参考 fcitx5 的方法修改。

服务端启动命令如下：

```bash
xpra start \
	:100 \
	--daemon=no \
	--env=GTK_IM_MODULE=fcitx \
	--env=QT_IM_MODULE=fcitx \
	--env=XMODIFIERS=@im=fcitx \
	--input-method=keep \
	--mdns=no \
	--start=fcitx5 \
	--start=firefox
```

简单的解释一下：

1. `:100` 表示会话编号，也就是 X 会话的 `DISPLAY` 环境变量。
2. `--daemon=no` 一般可以不用，这里是为了方便输出日志在终端方便查看。
3. `--env` 设置的环境变量用于 fcitx5 输入法，如果是 ibus 输入法，则需要设置对应的其他环境变量。`--input-method` 这里设置为 `keep` 表示 xpra 不会修改对应的环境变量，完全由我们指定。
4. `--mdns=no` 这里我们禁用，一般用不到。
5. `--start` 用于指定要启动的程序，可以使用多次来指定多个，这样方便我们把多个应用程序一起启动，放在同一个会话里方便使用。此外还有一个 `--start-child` 也可以指定要启动的程序，一般和 `--exit-with-children` 搭配使用，如果设置 `--exit-with-children=yes`，xpra 会在所有的 `--start-child` 退出的时候结束掉 xpra 自己。但是我们启动了输入法 fcitx5，一般都不会退出了，使用这个 `--start-child` 也没有什么必要。
6. 特别注意，输入法的切换键很可能容易与本地的 Windows 的输入法切换冲突，建议修改 fcitx5 的配置，设置输入法切换键为 CTRL + SHIFT。
7. 如果你的服务器没法使用 SSH 登录，但是有 TCP 端口可以访问，也可以用 `--bind-tcp=0.0.0.0:14500` 来允许通过 TCP 连接。
8. 如果的系统是 Ubuntu，xpra 对 snap 包的支持不是特别好。例如你的 firefox 是通过 snap 安装的，甚至你用 `apt install firefox` 实际上安装的还是 snap 版本的 firefox，参考 [issue #3740](https://github.com/Xpra-org/xpra/issues/3740)，你可能会遇到 `/user.slice/user-1000.slice/session-85.scope is not a snap cgroup`，然后实际上无法启动 firefox。一个解决方法是给 xpra 命令加上选项 `--dbus-launch=no`。最好的方法是不用 snap。
9. 部分应用程序是不允许启动多个实例的，例如你用 xpra 在 `:100` 启动了一个 firefox，就无法在 `:101` 启动新的 firefox。

然后是客户端连接，以 Windows 的客户端连接为例，启动 xpra 的客户端之后，点击 Connect，选择 Mode 为 SSH，填入用户名、IP、SSH 端口、DISPLAY 号，密码输入登录密码，如果配置了 SSH 公钥登录则无需填入密码，然后点击 Connect 即可。为了方便，用户可以把这个配置保存为文件，点击 Save 命令并选择合适的保存路径，保存为 `.xpra` 文件即可。注意高分屏下可能保存对话框太大没有显示出来保存按钮，需要调整一下对话框尺寸。下次使用的时候，直接双击打开这个 `.xpra` 文件即可自动填入对应的配置，无需每次填写配置。

类似的，如果前面服务端启动的时候用的 TCP，则在客户端这里选择 TCP，然后输入对应的信息即可。此外，如果服务端还安装了 [xpra-html5](https://github.com/Xpra-org/xpra-html5)，你还可以使用浏览器访问 `http://server-ip:14500`，直接在浏览器里连接，无需使用客户端。

这里我们没有设置 TCP 连接方式的认证方法，这是很不安全的，一般不建议使用。

xpra 实际上也支持直接启动一个桌面环境，只需要把 `xpra start` 换成 `xpra start-desktop`，然后在 `--start` 选项指定启动桌面的命令即可，例如 `--start=startlxqt`，其他的选项设置和 `xpra start` 基本类似。

特别注意，xpra 服务端只支持 X11，因此一般只能用于启动 lxqt, xfce4 等使用 X11 的桌面环境，Gnome 和 KDE 之类的是不支持的，即使他们可以设置使用 X11，一般也不会有很好的支持。

从节约资源的角度来说，我们完全没有必要去启动一个完整的桌面。而且如果我们需要在这个会话里启动更多的程序，可以在客户端状态栏的 xpra 图标点击 Start 然后选择对应的应用程序去启动。

如果我们需要断开连接，点击系统托盘处的 xpra 选择 Disconnect 即可，我们随时可以重新连接回来。

右下角托盘里的 xpra 还有很多功能选项，比如 Server 菜单下支持的文件互传等，就留给读者自己去探索了。

## 总结

本文简单介绍了 xpra 如何配置和使用，并且给出了一个实际可用的示例，该示例支持中文用户比较需要的 fcitx5 输入法等功能。用户可以根据自己的需要简单修改该命令，即可适用各种情况。

总的来说，xpra 提供了一个非常有特色的转发单个应用程序的远程连接方案，值得尝试。缺点在于部分应用程序的支持不算太好，例如 snap 版本的 firefox 之类的。不过实际上服务器用 Linux 的话，需要用到图形界面的程序实际上并不多，出了一些不得不用的，例如特定的专业软件 MATLAB 之类的。可惜我没有 MATLAB，也没有实际使用 MATLAB 的需要，无法做进一步的测试和验证。

## 参考

1. [Xpra环境中文输入](https://cloud-atlas.readthedocs.io/zh_CN/latest/linux/desktop/xpra/xpra_chinese_input.html)
2. [X持久化远程应用Xpra快速起步](https://cloud-atlas.readthedocs.io/zh-cn/latest/linux/desktop/xpra/xpra_startup.html)
