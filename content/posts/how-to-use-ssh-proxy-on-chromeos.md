---
title: "Chrome OS 下使用 SSH proxy 科学上网"
date: 2014-08-15T00:00:00+08:00
---

在大陆破网必然是没法好好的使用 Google 的资源的啦。既然买了 Chromebook，也用 Chrome OS，而不是安装其他系统。为了更加顺利的访问网络，一些必要的手段是不可少的。下面两个方法比较好的说： 

1. VPN
1. SSH

这里只说 SSH。其实也没什么好说，给两个链接：[Chromebook SOCKS Proxies and SSH Tunnels](http://blog.sahal.info/post/58278726443/chromebook-socks-proxies-and-ssh-tunnels)；[HOW TO USE SSH TUNNELING ON CHROME OS](http://hideki.hclippr.com/2011/07/24/how-to-use-ssh-tunneling-on-chrome-os/) 上面有个链接竟然失效了，不过没有关系，我发现有一种更加简单的方法咯。 首先要安装这个应用，[Secure Shell](https://chrome.google.com/webstore/detail/pnhechapfaindjhompbnflcldabbghjo)，然后启动它。填上连接 SSH 的必要信息，基本上就是 user@host:port，SSH Arguments 写上"-D 8080"，8080 是转发端口，可以根据需要修改。使用 SSH key 登录的可以在 Identity 那里导入 SSH 密钥对。最后点一下连接就可以了。SSH 私钥有密码加密的就按照提示输入即可。OK，这里就利用 Secure Shell 建立好了 SSH tunnel。Chrome 上再装个 [Proxy SwitchyOmega](https://chrome.google.com/webstore/detail/padekgcemlokbadohgkifijomclgjgif)，这个是 [Proxy SwitchySharp](https://chrome.google.com/webstore/detail/proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm) 的升级版，Proxy SwitchySharp 据闻已经不再维护。还没成功翻越万里长城的可以先去 GitHub 下载 [Proxy SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega/releases)。 做人要厚道，把参考链接也要放在这里的，[Chrome浏览器下的Secure Shell ssh client插件](http://weekend.blog.163.com/blog/static/7468958201302271444247/)
