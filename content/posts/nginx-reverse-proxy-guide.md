---
title: "使用Cloudflare和Nginx配置子域名反向代理与HTTPS访问的一些心得记录"
date: 2024-11-06T00:00:00+08:00
toc: false
images:
tags:
  - Cloudflare
  - HTTPS
  - Nginx
  - docker
  - 自定义域名
  - CDN
  - SSL证书
  - 域名解析
---

## 简介

我自己有一个域名，也就是本站．然后我还有一台有公网 IP 的服务器，上面运行了各种服务，有一些是可以通过网页访问的．但是我暂时都没有给这些网页服务绑定一个域名．为了方便使用，我可以用我自己的这个域名的子域名来绑定一下，方便我自己访问，就不用每次都使用 http://IP:port 的形式来访问了．此外，我还可以白嫖一下 Cloudflare 的服务，加上 HTTPS 访问．

## 前情提要

本站点是一个静态站点，实际上是 hugo 生成的静态站点，源码托管在 github pages 或者 gitlab pages 上，绑定了自定义的域名．同时用 Cloudflare 的服务来提供了 HTTPS 的访问．关于这一点如何操作，互联网上已经有很多教程和资料，这里不再赘述．

现在，我打算添加子域名解析到我要用的其他网页服务去，但是前提肯定是不能影响我这个主站的正常运行．

## 详细的步骤

为了方便描述，下面先列出一些假设：

1. 我的公网服务器 IP 为：1.1.1.1．
2. 我在公网服务器上部署了一个网页服务，可以通过 http://1.1.1.1:8080 访问到．
3. 我的域名为 example.org．
4. 我现在想要直接通过 https://web.example.org:8081 或者 http://web.example.org:8081 访问到 http://1.1.1.1:8080 的服务．

这里我们可以通过在服务器 1.1.1.1 上使用 Nginx 反向代理，监听 8081 端口，将来自 web.example.org 的请求转发给 http://1.1.1.1:8080，即可完成任务．

具体的，Cloudflare 的设置需要：

1. 在 Cloudflare 的 DNS 设置添加 A 记录，将 web.example.org 解析到 1.1.1.1，并勾选 proxy 选项．勾选 proxy 选项表示使用 Cloudflare 的 CDN 服务，不勾选则表示仅仅做 DNS 解析．我们想要白嫖 Cloudflare 的 CDN，且用上 HTTPS，所以这里要勾选．
2. 打开 Cloudflare 的 SSL/TLS->Origin Server，生成一个证书，颁发给 `*.example.org` 和 `example.org`．这个证书仅仅用于服务器 1.1.1.1 和 Cloudflare 之间的通信，有效期选择 15 年，免得更换．这里按照提示保存得到两个文件，分别记作 `example.org.key` 和 `example.org.pem`．
3. 检查一下 cloudfalre SSL/TLS 设置 encryption mode 为 flexible 或者 full．这个一般都是 ok 的．
4. 检查一下 Cloudflare 里 SSL/TLS 的 Edge Certificates 开启 Always Use HTTPS。这个选项是为了使用 Cloudflare 的 CDN 服务的时候，用户访问 http://web.example.org:8081 会自动跳转到 https://web.example.org:8081．这个选项一般也是已经开启了的．

1.1.1.1 服务器上的设置：

1. 这里我们使用 docker compose 来启动 Nginx 服务，Nginx 配置文件 `proxy.conf` 内容如下：

```
map $host $proxy_target {
    # 域名和代理目标服务器的映射
    web.example.org http://1.1.1.1:8080;
    # 默认设置一个空的代理目标服务器
    default "";
}

server {
    listen 443 ssl;
    # 服务器名称，也就是域名
    server_name web.example.org;
    # HTTPS 证书，仅用于源服务器与 Cloudflare 之间的通信
    # 由 Cloudflare 签发给 *.example.org，有效期15年
    # 完全不用更新了
    ssl_certificate /etc/nginx/cert/example.org.pem;
    ssl_certificate_key /etc/nginx/cert/example.org.key;

    location / {
        # 检查代理目标服务器的这个变量，为空则直接返回 404
        if ($proxy_target = "") {
            return 404;
        }
        # 代理目标服务器
        proxy_pass $proxy_target;
        # 设置请求头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 600s;
        proxy_connect_timeout 600s;
        proxy_buffering off;
    }
}
```

这里我们使用了 Nginx 配置里的 `map`，方便后续根据需要添加更多的子域名．只需要添加子域名和对应的反向代理目标服务器即可．

`docker-compose.yaml` 文件内容如下：

```yaml
name: nginx-proxy
services:
  nginx:
    image: nginx:latest
    container_name: nginx_proxy
    ports:
      - "8081:443"
    volumes:
      - ./proxy.conf:/etc/nginx/conf.d/proxy.conf:ro
      - ./cert/example.org.pem:/etc/nginx/cert/example.org.pem:ro
      - ./cert/example.org.key:/etc/nginx/cert/example.org.key:ro
```

2. 启动服务：`docker compose up -d`．

现在，用户访问 `http://web.example.org:8081` 会自动跳转到 `https://web.example.org:8081`，最后实际上访问到 `http://1.1.1.1:8080` 提供的网页服务了．当然，用户访问 `https://web.example.org:8081` 也会是访问到 `http://1.1.1.1:8080` 提供的网页服务．

这样就实现了用户（浏览器）访问我们的站点的时候用上了 HTTPS 了，提高了安全性．比如说你的网页可以使用语音输入，浏览器会要求只能在 HTTPS 下使用麦克风设备，如果直接通过 IP 访问，就得自签名一个证书，还会看到浏览器的不安全警告．

## 进一步的改进

但是有的时候事情并不尽如人意．如果我们有一个新的网页服务部署在 `http://1.1.1.1:8090`，但是他的前端代码里有一些目标为 HTTP 站点的请求，例如他可能发送请求给 `http://2.2.2.2:8080/api/` 获取某个响应结果．这个时候继续按照上面的方法去操作，对于浏览器来说，就是要在 HTTPS 站点里发起了对 HTTP 目标服务器的请求了，也就是 HTTP over HTTPS，这个是不安全的．特别是这种从 HTTPS 站点主动请求 HTTP API 的情况，一般都会被浏览器直接禁止，从而导致这个网页服务无法使用．

最好的情况当然是我们对 `http://1.1.1.1:8090` 上的服务有完全的掌控力，可以去修改其中的代码．具体的修改可以是改成请求到 HTTPS 的接口，如果外部服务不提供 HTTPS 的接口请求，还可以自己再做一个代理服务器来转发．如果无法修改，那我们只好继续使用 http 访问了．这个时候配置会略有不同．

具体的，假设我们想要使用 `http://vip.example.org:8091` 访问 `http://1.1.1.1:8090`，Cloudflare 的配置：

1. 在 Cloudflare 的 DNS 设置添加 A 记录，将 web2.example.org 解析到 1.1.1.1，并且取消勾选 proxy 选项．这里我们只需要做 DNS 解析．

1.1.1.1 服务器上的设置：

1. Nginx 配置文件 `proxy.conf` 新增一个 server 块：

```
server {
    listen 80;
    # 服务器名称，也就是域名
    server_name vip.example.org;

    location / {
        # 代理目标服务器
        proxy_pass http://1.1.1.1:8090;
        # 设置请求头
        proxy_set_header Host $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 600s;
        proxy_connect_timeout 600s;
        proxy_buffering off;
    }
}
```

然后 `docker-compose.yaml` 中新增一个端口映射，完整配置如下：

```yaml
name: nginx-proxy
services:
  nginx:
    image: nginx:latest
    container_name: nginx_proxy
    ports:
      - "8081:443"
      - "8091:80"
    volumes:
      - ./proxy.conf:/etc/nginx/conf.d/proxy.conf:ro
      - ./cert/example.org.pem:/etc/nginx/cert/example.org.pem:ro
      - ./cert/example.org.key:/etc/nginx/cert/example.org.key:ro
```

启动这个 Nginx 反向代理服务之后，用户访问 `http://vip.example.org:8091` 即可对应访问到 `http://1.1.1.1:8090` 的服务了．

## 补充

关于公网服务网 `1.1.1.1` 的端口，如果你的 80 和 443 端口可以使用，那就可以不用非标端口 `8091` 和 `8081`．这样用户访问的域名后面也就不用加上端口了．不过如果你的服务器在国内的话，一般 80 和 443 端口都是被云服务商的防火墙挡住的，必须备案后才能使用．有的云服务上即使你使用了非标端口，如果域名没有备案，也是不允许通过域名访问的，但是并不限制通过 IP 加端口的方式来访问．此时，你只需要删掉 Nginx 配置文件中的 `proxy_set_header Host $remote_addr;` 即可．也就是不在请求头里设置 `Host` 为你的域名，云服务上就不知道你这个请求是从一个域名解析过来的，然后就放行了．
