---
title: "正确安装 opencv 包并支持 H264 编解码的姿势"
date: 2023-11-14T00:00:00+08:00
toc: false
images:
tags: 
  - opencv
  - anaconda
  - docker
  - python
---

## 起因

这篇文章的起因是我需要安装 Python 包 opencv，且需要支持 H264 编解码．我有一个项目会使用 opencv 读写视频，并且这个视频需要能够在网页上播放．一般 opencv 写视频文件的时候会用 "XVID"，但是这个在很多浏览器上无法直接播放．

用于一般容易遇到的错误提示是：

```text
OpenCV: FFMPEG: tag 0x34363268/'h264' is not supported with codec id 27 and format 'mp4 / MP4 (MPEG-4 Part 14)' OpenCV: FFMPEG: fallback to use tag 0x31637661/'avc1' Could not find encoder for codec id 27: Encoder not found
```

这个主要是因为他们从 PyPI 安装了 opencv-python 这个包，但是这个包由于许可证的原因，并不包含 H264 编解码器的支持．

## 解决方法

其实说到这里，解决方法已经呼之欲出了．用户只需要从 Anaconda 安装 opencv 即可，那个是带有 H264 编解码功能的．以 conda 虚拟环境为例子：

```bash
conda create -n opencv python=3.10 -y
conda activate opencv
conda install opencv!=4.6 -y
# 也可以从 conda-forge 安装，应该是一样的
# conda install opencv!=4.6 -y -c conda-forge
```

这里再特别说明一下，不安装 opencv 4.6.x 版本是因为该版本有一个 [bug](https://github.com/opencv/opencv/issues/22088) 会导致读取元信息中包含旋转的视频的时候会出错．就本文写作的时间，这个实际安装的版本就是 4.5.5 版本，同时又因为 4.5.5 版本最高只为 python 3.10 打包，所以这里创建的环境是 python 3.10 的．

按理说，到这里就应该是解决了问题了．但是有一些包在写依赖的时候指明了依赖 opencv-contrib-python，这个包是来自 PyPI 的．如果你使用 pip 命令安装一些包的时候不小心把这个包作为依赖安装进来了，就有可能导致你在使用 `cv2.VideoWriter` 的时候实际上使用的是 opencv-contrib-python，而非 Anaconda 中的 opencv，而前者是不带 H264 编解码器支持的．最终导致你命名安装了 Anaconda 的 opencv，但是依然出错．

建议先检查当前环境中的 opencv 相关的来自何处，如果 `conda list | grep opencv` 看到有来自 PyPI 的 `opencv-contrib-python`，只需将其卸载掉即可：

```bash
pip uninstall opencv-contrib-python
```

真的如此么？如果你仔细查看，他会提示你这个卸载将会移除 `site-packages/cv2/*`，这样就会把 conda 安装好的 opencv 包也给卸载了．

### 可能的解决方案

1. 先安装好其他包，然后卸载掉来自 PyPI 的 opencv-contrib-python，最后再用 conda 安装 opencv．
2. 确认好依赖 opencv-contrib-python 的包，例如 mediapipe，然后使用 `pip install --no-deps mediapipe` 安装 mediapipe，但是不安装他的依赖包．接着使用 conda 安装 opencv．最后再根据代码实际运行时候的 Import 错误提示来安装其他包．这里也可以使用 `pip show mediapipe` 命令查看这个包的依赖，然后手动安装其中除了 opencv-contrib-python 之外的包．
3. 如果你需要安装的包编译比较简单，也可以下载源码，修改其依赖关系，然后编译安装．
4. 对提供二进制 wheel 文件的，你还可以将其 wheel 文件下载之后修改其中的依赖关系文件重新打包，然后手动安装重新打包的 wheel 文件．
  4.1. 下载这个 wheel 文件，但是不下载他的依赖包，`pip download --no-deps mediapipe`．
  4.2. 解压 wheel 文件，`wheel unpack FILENAME`．
  4.3. 编辑 `METADATA`，删除依赖 opencv 的行．
  4.4. 重新打包 wheel 文件，`wheel pack WHL-DIR`．
  4.5. 最后使用 `pip instsall FILENAME` 安装即可．此时就不会再去安装 opencv 这个依赖．

以上的这些解决方案都不能算是完美，毕竟在 conda 虚拟环境中使用 pip，相当于是在混用 conda 和 pip 两个包管理工具，难免会出现不兼容的问题．

无意之中踩坑，挣扎许久，记录下来，希望对后来的人有用．
