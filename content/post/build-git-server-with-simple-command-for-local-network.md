---
title: "几行命令在 Arch Linux 下为本地局域网搭建一个 git server"
date: 2017-11-24T00:00:00+00:00
---

## 安装 git

So easy,

```shell
pacman -S git
```

## 启动 `git-daemon.socket`

```shell
systemctl start git-daemon.socket
systemctl enable git-daemon.socket
```

这个 daemon 的实际启动命令为：

```shell
ExecStart=-/usr/lib/git-core/git-daemon --inetd --export-all --base-path=/srv/git
```

从这里可以知道我们的 git 仓库放在 `/srv/git` 目录下．只要在该目录下创建一个 git 仓库，它就会自动被 git server 管理．用户访问该仓库的命令类似这样：

```shell
git clone git@IP:/srv/git/repository.git
```

其中 `IP` 为 git server 的 IP，`repository` 为仓库的名称．

其实到这里就已经搞定了，是不是很简单？不是啦，还是需要一些配置的．

## 添加访问权限

如果你真的使用上面的命令去 clone 一个仓库，肯定是没有权限的．我们需要添加 ssh key．一般我们把 ssh 公钥放到 `~/.ssh/authorized_keys` 里去．咦？`git` 用户的 `$HOME` 在哪里呢？用这个命令查看

```shell
eval echo "~git"
```

结果是 `/`．好像直接在根目录下放不太好吧．哦，那就修改一下 git 用户的 home 目录啊．

```shell
usermod --home /home/git git
```

然后把用户的公钥添加到 `/home/git/.ssh/authorized_keys` 里去就好啦，多个用户的话就一行添加一个公钥．这下就可以愉快地使用了．

