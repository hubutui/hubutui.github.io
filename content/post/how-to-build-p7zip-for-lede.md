---
title: "如何为 LEDE/OpenWRT 打包 p7zip"
date: 2017-08-04T00:00:00+08:00
---

p7zip 是移植到 POSIX/Unix-like 系统的 7-Zip 软件，可以压缩解压 7z 格式的文件．这个一款非常优秀的开源软件．LEDE/OpenWRT 也是用的 Linux 内核，那么 p7zip 自然也是可以在上面运行的．只可惜官方并没有打包 p7zip．若是需要解压或者压缩 7z 格式的压缩包，难免有些不便．既然如此，也只好自己动手，丰衣足食了．

## 安装 SDK

直接到 LEDE 主页或者镜像网站下载对应路由器的 SDK，以联想 newifi 为例，我们应该下载 [这个 SDK](http://mirrors.ustc.edu.cn/lede/releases/17.01.2/targets/ramips/mt7620/lede-sdk-17.01.2-ramips-mt7620_gcc-5.4.0_musl-1.1.16.Linux-x86_64.tar.xz)．下载后直接解压即可．

### SDK 的配置

```bash
cd $sdkdir
make menuconfig
```

其中 `$sdkdir` 为 LEDE SDK 的目录．进入 `Global build settings`，将里面的四个选项全部取消勾选，按 `ESC` 退出并保存．

## 编译打包 p7zip

请直接参考 [p7zip-lede](https://github.com/hubutui/p7zip-lede)．需要说明的是，
这里打包的是最全的 `7z`，如果你只需要 `7za` 或者 `7zr`，比如说你的路由器存储空间不足，无法安装完整的 `7z`，那么你需要根据参考 `p7zip` 的 `README` 修改 `Makefile` 中的 `MAKE_FLAGS` 等内容．

## 另一种解压 7z 文件的方法

如果仅仅需要解压 7z 文件，还有另外一种方法，并不是非 p7zip 不可．我们还可以选择安装 bsdtar：

```bash
opkg update
opkg install bsdtar
```

然后直接用 `bsdtar` 解压 7z 文件即可．`bsdtar` 支持解压 tar, pax, zip, xar, lha, ar, cab, mtree, rar, warc, 7z 格式的压缩包以及 ISO 镜像，可以创建 Writes tar, pax, zip, xar, ar, ISO, mtree 以及 shar 格式的压缩包，并能够自动处理 gzip, bzip2, lzip, xz 和 lzma 等常用的压缩算法．足以应付无线路由器上用得到的各种压缩包了．

可惜的是 `bsdtar` 不支持解压加密了的 RAR 压缩包，偶尔还是会有不便．

## 参考

* [LEDE 论坛上的讨论](https://forum.lede-project.org/t/how-to-compile-p7zip-to-lede-openwrt/5079/18)
* [一个用于 OpenWRT 的 Makefile](https://github.com/tobiaswaldvogel/openwrt-addpack/blob/master/p7zip/Makefile)
* [LEDE SDK](https://lede-project.org/docs/guide-developer/compile_packages_for_lede_with_the_sdk)
* [LEDE Build System](https://lede-project.org/docs/guide-developer/install-buildsystem)
* [p7zip-lede](https://github.com/hubutui/p7zip-lede)
