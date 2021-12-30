---
title: "XRDP 相关的一个小问题——关于 `thinclient_drives` 目录的错误"
date: 2018-01-16T00:00:00+08:00
---

关于 [XRDP](http://www.xrdp.org) 是什么，我就不废话了．它会在远程 Linux 主机的用户家目录下创建一个 `thinclient_drives` 目录，用于驱动器的映射．这个目录实际上应该是用 `fuse` 挂载的．但是有的时候会遇到该目录的权限错误，如下：

```text
d????????? ? ?        ?         ?             ? thinclient_drives/
```

它的权限位全部变成了问号．这个是因为挂载出现了问题．如果你尝试使用 `fuser thinclient_drives` 去分析它，会得到提示：

```text
无法分析 /home/user/thinclient_drives: 传输端点尚未连接
```

或者

```text
Cannot stat /home/user/thinclient_drives: Transport endpoint is not connected
```

实际上就是挂载出现了错误．如果你使用 XFCE 的文件管理器 Thunar，你甚至会无法打开文件管理器．解决方法很简单，只需要将其正确卸载即可：

```shell
fusermount -u thinclient_drives
```

Over.

参考：[FUSE error: Transport endpoint is not connected](https://stackoverflow.com/questions/16002539/fuse-error-transport-endpoint-is-not-connected)
