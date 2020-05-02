---
title: "Windows下使用MSYS2进行开发小记录"
date: 2020-05-02T00:00:00+08:00
---

## MSYS2是什么？

根据[官网](https://www.msys2.org/)的说法，MSYS2是一个为Windows打造的软件发行版与构建平台。MSYS2基于Cygwin和MinGW-w64构建。它提供了类似Linux下的基本bash的一个环境，同时也让用户更加方便地使用MinGW-w64（其实主要就是GNU GCC那一整套）开发原生的Windows应用。

MSYS2的魅力在于我们可以在Windows下获得类似Linux下的终端的生态系统，从而可以把Linux下的软件移植过来，或者直接开发跨平台的软件。另外，它采用pacman作为包管理工具也是极具特色。使用pacman的话，我们可以很容易地使用写个PKGBUILD文件来完成软件包的打包工作。开发者可以很容易地使用PKGBUILD来进行打包工作，这其实也是ArchLinux比其他Linux发行版更为吸引人的地方。毕竟，我们不仅可以从AUR上获取大量可以直接使用的PKGBUILD来自己打包，还可以照猫画虎自己打包。ArchLinux的包是个人见过最简单的最容易打包的了，比RPM包和Deb包都要简单得多。

## 如何安装

建议直接从国内镜像网站下载和安装，比如[清华镜像](https://mirror.tuna.tsinghua.edu.cn/help/msys2/)，[中科大镜像](https://mirrors.ustc.edu.cn/help/msys2.html)。安装嘛，也就是下载安装包打开之后一路按照提示下一步即可。链接中还给出了配置软件源的方法，建议按照提示替换为国内的源，下载速度更快，也更稳定。温馨提示，在MSYS2中使用pacman更新系统（指MSYS2）之后，可能会将软件源替换为默认的，若发现速度异常，应该再次检查软件源。

## 如何使用

其实这个没事好说的啊，软件包的管理需要用pacman，这个的用法可以参考ArchLinux的[wiki](https://wiki.archlinux.org/index.php/Pacman)，或者MSYS2的[文档](https://www.msys2.org/wiki/Using-packages/)。这里值得一提的是，MSYS2把软件仓库分为了msys2、mingw32和mingw64三个。其中msys2仓库中的软件包名字与Linux下对应的名字是一致的，除了部分MSYS2特有的包。mingw32仓库的软件包名字加了`mingw-w64-i686-`前缀，表示它是32位的软件包。类似的，mingw64仓库的软件包名字加了`mingw-w64-x86_64`，表示它是64位的软件包。当你给MSYS2打包一个新软件包或者开发一个新软件包的时候，你就需要考虑一下应该把它放到哪个仓库去了。msys2软件包是依赖于`msys-2.0.dll`的软件包，大部分时候你在Windows下用MSYS2开发的包都不属于这一类。原生的Windows程序不依赖`msys-2.0.dll`，他们一般是依赖Windows的`msvcrt.dll`。

msys2仓库中所有包的`PKGBUILD`文件托管在代码仓库[MSYS2-packages](https://github.com/msys2/MSYS2-packages)中，目前大约有500个包。而mingw32和mingw64仓库中所有包的`PKGBUILD`托管在代码仓库[MINGW-package](https://github.com/msys2/MINGW-packages)中，目前大约有1400个包。

## 更换默认的SHELL为zsh

这个其实不难，只是有一个点网上搜来的东西都没说得太清楚。首先，你需要安装`zsh`和`msys2-launcher-git`。然后需要在配置文件`/msys2.ini`、`/mingw64.ini`和`/mingw32.ini`中加入下面这句：

```
SHELL=/usr/bin/zsh
```

并且，此后启动MSYS2终端的时候应该打开MSYS2安装目录下的`msys2.exe`、`mingw64.exe`或`mingw32.exe`。这样启动后的终端默认就是用zsh为SHELL了。有需要的用户可以自己安装oh my zsh，按照官网提示下载oh my zsh到本地，修改`~/.zshrc`文件即可。

略微可惜的是，MSYS2的zsh的路径补全有一点小问题。MSYS2把Windows的各个盘符挂载到了根目录下的同名目录，也就是把C盘挂载到`/c`，把D盘挂载到了`/d`目录。但是`zsh`无法补全类似`/d/testdir`这样的路径，而`bash`是可以做到的。

## 自己打包

这部分直接参考官方[文档](https://www.msys2.org/wiki/Creating-Packages/)即可，个人建议直接抄现成的`PKGBUILD`。特别需要注意以下几点：

1. 使用msys终端（`msys2.exe`）而非64bit终端（`mingw64.exe`或`mingw32.exe`）来执行打包命令 `makepkg`和`makepkg-mingw`。`makepkg`用于创建msys软件包，而`makepkg-mingw`则打包mingw64和mingw32的原生Windows软件包。
2. 你在`PKGBUILD`中看到的一些特殊的变量，如`MINGW_PREFIX`、`MINGW_PACKAGE_PREFIX`之类的，它们的定义保存在`/etc/makepkg.conf`、`/etc/makepkg_mingw64.conf`和`/etc/makepkg_mingw32.conf`这三个文件中。其意思参考`PKGBUILD`中的用法和它的定义应该可以很容易理解。
3. 当然，打包的工具链的话，你至少需要`base-devel`、`msys2-devel`（msys2包）、`mingw-w64-i686-toolchain`（mingw32包）和`mingw-w64-x86_64-toolchain`（mingw64包）。

### 部署和分发

假设你现在编译好了你的软件包`foo`，想要把它部署到一台没有安装MSYS2的机器上，或者想要分发给没有MSYS2的用户使用，那么你大概可以这么操作。一般的，我们使用动态链接的方式来编译我们的软件包，那么我就需要保证程序运行的时候可以找到这些动态链接库。因此，我们需要将这些动态链接库与我们编译好的二进制文件一同发布即可（暂不考虑程序运行时需要的其他资源文件）。要找到这些动态链接库，我们可以使用 `ntldd` 或者 `ldd` 命令来查找。例如，`ldd`告诉我编译好的这个`foo`文件链接了这些库：

```text
        ntdll.dll => /c/Windows/SYSTEM32/ntdll.dll (0x7ffe96740000)
        KERNEL32.DLL => /c/Windows/System32/KERNEL32.DLL (0x7ffe961e0000)
        KERNELBASE.dll => /c/Windows/System32/KERNELBASE.dll (0x7ffe93930000)
        msvcrt.dll => /c/Windows/System32/msvcrt.dll (0x7ffe962a0000)
        libstdc++-6.dll => /mingw64/bin/libstdc++-6.dll (0x6fc40000)
        libgcc_s_seh-1.dll => /mingw64/bin/libgcc_s_seh-1.dll (0x61440000)
        libwinpthread-1.dll => /mingw64/bin/libwinpthread-1.dll (0x64940000)
```

那我们只需要将`/ming64`目录下的那几个库都跟`foo`一起打包即可。其他在C盘的动态链接库应该是Windows自带的，可以忽略。或者保险起见，一并打包。

## 小结

使用MSYS2可以很方便地让我们得到类似Linux下的开发体验。MSYS2提供了大量优秀地跨平台的包和库等等，比如opencv，ITK, VTK等等，我们都可以直接使用，而不要自己折腾编译了。即使我们不直接使用MSYS2中的这些包，我们依然可以参考它提供的`PKGBUILD`文件来自己编译。大概就这么多吧，有机会再来写怎么用Visual Studio Code来结合MSYS2愉快地工作吧。
