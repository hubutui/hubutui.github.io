---
title: "利用 KDE Connect 将你的手机屏幕改造成笔记本的触摸板"
date: 2014-08-17T00:00:00+08:00
---

前段时间就有看到微博上的这个消息，KDE Connect 可以将手机屏幕当成触摸板来用！这简直就是黑科技啊。今天亲测发现 KDE Connect 果然不是一般的神器啊。

## KDE Connect 是什么？

KDE Connect 致力于连接您的所有设备。比如说，你可以在电脑上查看手机上的通知，共享两端的剪切板；或者把手机当成一个遥控器来控制你的电脑，播放音乐；甚至，你还可以把你的手机当成触摸板，从此抛弃鼠标！ KDE Connect 是 Albert Vaca 参加2013年的[谷歌编程之夏](http://code.google.com/soc/)的成果。这还是我第一次感受到谷歌编程之夏带来的好处诶。 

## 安装 KDE Connect

下面就来说一下要怎么安装 KDE Connect 了。 

### 电脑端

在 openSUSE 下，KDE Connect 的软件包名为 `kdeconnect-kde`，位于 `KDE:Extra` 源里。你可以在[这个页面](https://build.opensuse.org/project/repositories/KDE:Extra)根据对应的系统版本添加软件源。添加好软件源之后，刷新软件源：

```bash
zypper refresh
```

然后安装 kdeconnect-kde 软件包：

```bash
zypper in kdeconnect-kde
```

#### 配置 kdeconnect-kde

安装好 kdeconnect-kde 之后还需要配置一下。首先要激活 kdeconnect-kde，使用非 root 用户执行以下命令：

```bash
$ qdbus org.kde.kded /kded loadModule kdeconnect
```

如果执行成功会返回 `true`。 然后，刷新配置：

```bash
kbuildsycoca4 -noincremental
```

现在，你打开「系统设置」->硬件->KDE Connect 就可以看到 KDE Connect 的配置了。目前这里还是空白，因为还没有和手机配对连接。 

##### 防火墙配置

KDE Connect 使用的端口范围为1714~1764，为此我们还需要打开相应的端口以便 KDE Connect 能够连接手机与电脑。具体操作为，打开YaST->安全和用户->防火墙->允许的服务，添加 HTTP Server；选中 HTTP Server，点高级，TCP 端口以及 UDP 端口都写上 1714:1764 即可。 电脑端的配置算是完成了。接着是手机/平板端。 

### 手机/平板端

KDE Connect 目前支持安卓手机和平板，苹果，Windows Phone 以及黑莓不详。这里只说安卓手机，要求安卓4.1以上，低版本可能可以用，但是估计有功能缺陷。 你可以从 [Google Play](https://play.google.com/store/apps/details?id=org.kde.kdeconnect_tp) 或者 [F-Droid](https://f-droid.org/repository/browse/?fdid=org.kde.kdeconnect_tp) 下载安装 KDE Connect 到你的手机上。 

## 连接手机与电脑

用 KDE Connect 连接手机与电脑是很方便的。 首先，需要确保两台机器在同一个局域网下。然后手机端打开 KDE Connect，你就可以在 NOT PAIRED DEVICES 下看到你的电脑了，点一下，然后 KDE Connect 会提示说该设备未配对，点一下 Request pairing 就可以向电脑发出配对请求了。这时候电脑通知区域就会弹出 KDE Connect 的配对通知了，点一下 Accept 即可配对成功。 另外一种配对方式是从电脑发出配对请求，只需要打开「系统设置」->硬件->KDE Connect 点一下就好，和手机端的操作方法大同小异，没什么好说的。 

### 配对之后能干嘛？

目前 KDE Connect 支持的特性如图所示：

![KDE Connect](https://github.com/darcyhuu/darcyhuu.github.io/raw/master/img/KDE-Connect.png)

可以发现，目前来说，KDE Connect 支持手机与电脑共享剪切板，复制粘贴会变得很方便；在电脑上浏览手机的文件（需要 sshfs 支持）；发送和接受文件；互 ping 以检查是否连通；在手机上控制电脑端的媒体播放器，如 VLC 等；将手机屏幕当成触摸板来使用等等。 特别要说一下把手机屏幕当成触摸板这一点。手机上 KDE Connect 点开 Open touchpad control 即可把手机屏幕当成触摸板来使用。实测发现其支持单击，双击，三击，双指滑动等常用的操作，使用起来也非常的灵敏流畅。 至于互传文件，手机上选中需要发送的文件，然后分享/发送，选择 KDE Connect，即可发送文件到电脑；而电脑端发送文件也只需要选中文件，右键，选择 Send to '手机名称' via KDE Connect，非常的简单。在电脑上浏览手机的文件则只需要安装好 sshfs，然后打开 dolphin，左边的位置那里就有你的手机，直接点开就可以浏览手机上的文件了。 新的特性还在不断的完善之中，相信 KDE Connect 会越来越用的说。 

## 参考

* kde wiki：[KDEConnect](https://community.kde.org/KDEConnect)
* 开发者博客：[albertvaka.wordpress.com](http://albertvaka.wordpress.com/)
* 安装 KDE Connect：[How to integrate Android into KDE Linux desktop](http://xmodulo.com/2014/01/integrate-android-kde-linux-desktop.html)
* CloudPen 的博文：[KDE Connect，连接KDE和Android的理想工具](http://zhuyalin.cn/2014/08/02/kde-connect%EF%BC%8C%E8%BF%9E%E6%8E%A5kde%E5%92%8Candroid%E7%9A%84%E7%90%86%E6%83%B3%E5%B7%A5%E5%85%B7/)
* KDE Connect 的开发仓库：[kdeconnect-android](https://projects.kde.org/projects/playground/base/kdeconnect-android/repository), [kdeconect-kde](https://projects.kde.org/projects/playground/base/kdeconnect-kde/repository)

