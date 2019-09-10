---
title: "ArchLinux 上搭建 Jupyter Notebook 服务器"
date: 2018-09-01T00:00:00+08:00
---

## 安装 Jupyter 和 nginx

一行代码搞定：

```shell
# pacman -S jupyter nginx
```

启用 nginx 服务：

```shell
# systemctl start nginx
# systemctl enable nginx
```

## 配置

这个才是我想说的重点．看了一些资料，因为对 nginx 不是很了解，所以才看不明白．首先，为了避免升级的时候配置文件被覆盖，先创建两个存放配置文件的目录，`/etc/nginx/sites-available` 和 `/etc/nginx/sites-enabled`，有的发行版已经有了这两个目录的就可以不用创建了．一般我们将配置文件写到 `/etc/nginx/sites-available` 目录下，然后需要启用的时候在 `/etc/nginx/sites-enabled` 新建一个相应的软链接即可．编辑 `/etc/nginx/nginx.conf`，在 `http` 这一节加入一行：

```text
include /etc/nginx/sites-enabled/*
```

很多文章就只说 nginx 的配置很简单，但是不懂 nginx 的人看了还是摸不着头脑．实际上还要添加 `/etc/nginx/sites-available/jupyter.conf`：

```text
server {
    listen PORT; # PORT 为反代端口，根据需要选择
    location / {
        proxy_pass http://127.0.0.1:JUPYTER_PORT; # JUPYTER_PORT 为 Jupyter 运行端口
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_redirect off;
    }
}
```

要启用 Jupyter 的 nginx 配置的时候就应该新建一个软链接，这个才真的是很简单，略．

Jupyter 的配置也省略不写，无非是设置端口和密码登录等等，网上很容易搜索到的．

## 使用

使用的时候，首先启动 Jupyter Notebook，

```shell
jupyter notebook
```

然后如果是在本地，就直接点开终端输出的链接地址就好了．远程访问就将链接中的地址替换为对应的 IP，端口换成反代端口．实际上就是 `http://IP:PORT` 嘛，token 那部分可以省略，自己输入密码登录．

## 局域网使用

如果你只是想在局域网里使用，完全没有必要使用 Nginx 的反向代理，只需在启动 Jupyter Notebook 的时候指定一下 `--ip` 即可：

```shell
jupyter notebook --ip 0.0.0.0
```

然后就可以从局域网内的其他机器访问 Jupyter Notebook 了．

## 参考

[Arch Linux 搭建 Jupyter notebook 服务器](https://nwindy.moe/linux-jupyter-setup/)
