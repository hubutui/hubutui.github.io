---
title: "操蛋的 Ubuntu"
date: 2018-07-12T00:00:00+08:00
---

今天尝试着升级 Ubuntu，从 Ubuntu 14.04 LTS 升级到 Ubuntu 16.04．首先编辑 `/etc/update-manager/release-upgrades`，修改为：

```text
Prompt=lts
```

然后执行升级命令：

```shell
sudo do-release-upgrade
```

一切都很顺利的样子．登录上去执行一个 `sudo apt update`，提示 GLIBC 有误．明显就是 `libstdc++6` 这个包出问题了，直接从镜像源下载一个，然后使用 `dpkg -i package.deb` 安装．Done．然后再次执行 `sudo do-release-upgrade`，提示没有可以更新的版本．应该使用 `sudo do-release-upgrade -d`．结果更糟糕，提示遇到无法解决的问题，建议我报告 bug．不死心，修改 `/etc/update-manager/release-upgrades`：

```text
Prompt=normal
```

然后更新到了 17.10．再次尝试更新，`sudo do-release-upgrade`，依然提示同样的错误．Emm，肿么办？不怕，我们直接修改 `/etc/apt/sources.list`，把源换成 18.04，然后刷新软件源并升级：

```shell
sudo apt update
sudo apt dist-upgrade
```

终于给个有用的提示了，virtualbox 的一个包 virtualbox-dkms 有依赖问题无法解决．所以 do-release-upgrade 是二傻子么？遇到升级的时候遇到无法解决的依赖也不提示一下用户是什么依赖无法解决？先卸载 virtualbox，然后 `sudo do-release-upgrade`，这就可以升级了．

Ubuntu 这是自作主张，觉得用户没法解决问题，do-release-upgrade 根本就不给出有用的错误信息．Fuck you, Canonical Ubuntu！

要不是大家搞深度学习都用的 Ubuntu，我才不会考虑用它啊．真要用 Deb 系的也是考虑 Debian 啊．

