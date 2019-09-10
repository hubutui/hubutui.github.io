---
title: "利用 cURL 分块下载大文件"
date: 2018-01-20T00:00:00+08:00
---

## 缘起

自己的 VPS 是个小家伙，硬盘只有 10G，装完系统，能用的空闲空间也不过 7G 左右．但是我需要从国外的服务器下载一些数据，文件都比我的硬盘大．直接下载到本地速度不够快，而且下载链接有时效，只能考虑先下载到 VPS 上，然后再转移到本地．自然而然的想到一个办法就是分段下载文件到 VPS 上，最后转移到本地之后再拼接起来．

## 前提

使用本文的方法需要服务器支持 [HTTP Range Request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)，否则只能老老实实地买个硬盘大的 VPS 进行下载．至于怎么确认服务器是否支持，我首先想到的是从服务器分段下载一个较小的文件，然后拼接，对比直接下载完整的文件是不是一致．

其实也可以用 cURL 查看返回的请求头．例如：

```shell
curl -I http://mirrors.ustc.edu.cn/debian-cd/current/amd64/iso-cd/debian-mac-9.3.0-amd64-netinst.iso
```

返回结果

```
HTTP/1.1 200 OK
Server: openresty
Date: Sun, 21 Jan 2018 13:45:58 GMT
Content-Type: application/octet-stream
Content-Length: 307232768
Last-Modified: Sat, 09 Dec 2017 13:04:52 GMT
Connection: keep-alive
ETag: "5a2bdf74-12500000"
Accept-Ranges: bytes
```

看到了 `Accept-Ranges: bytes`，说明这个服务器是支持 HTTP Range Request 的．如果返回结果中不含 `Accept-Ranges`，则很有可能是不支持 HTTP Range Request 的．有的服务器会显式返回 `Accept-Ranges: none` 表示不支持 HTTP Range Request．

## 下载

常用的下载工具也就 wget, aria2, axel, curl 等几个．但是只有 curl 支持分段下载（注意，这里说的不是断点续传）．curl 有一个 `--range` 选项，可以指定下载文件的某一段．所以，下载命令大概就是这样子的：

```shell
curl --range 0-5000000000 -o part1 <url>
curl --range 5000000001- -o part2 <url>
```

这里是将文件分成两段下载，第一段的大小为 5000000000 字节，将近 5G 的样子，正好可以让我的 VPS 下载．

## 合并／拼接

将文件下载到本地之后，需要将其拼接起来，拼接文件可以使用 cat 或者 dd 命令．cat 据说速度会比较慢，而且也没有进度条，但是用法比较简单．

```shell
cat part1 part2 > outputfile
```

请注意输入文件的顺序不要搞错了．如果使用 dd，

```shell
dd if=part1 of=outputfile bs=4M conv=notrunc oflag=append status=progress
dd if=part2 of=outputfile bs=4M conv=notrunc oflag=append status=progress
```

其实上面的第一条命令可以用 `cp part1 outputfile` 替代，据说对于复制来说， cp 比 dd 要快．稍微解释一下使用到的选项，`bs=4M` 表示每次读写的块大小为 4M，`oflag=append` 表示将文件追加到输出文件的末尾，`conv=notrunc` 表示不截断文件，`status=progress` 表示显示进度．

## 缘灭

最后，确认合并的文件正确无误就可以删除掉分段文件了．

## 参考

* [How to Split and Download a Large File with cURL](https://www.maketecheasier.com/split-download-large-file-curl/)
