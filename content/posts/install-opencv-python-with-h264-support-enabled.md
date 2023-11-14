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
conda install opencv -y
# 也可以从 conda-forge 安装，应该是一样的
# conda install opencv -y -c conda-forge
```

按理说，到这里就应该是解决了问题了．但是有一些包在写依赖的时候指明了依赖 opencv-contrib-python，这个包是来自 PyPI 的．如果你使用 pip 命令安装一些包的时候不小心把这个包作为依赖安装进来了，就有可能导致你在使用 `cv2.VideoWriter` 的时候实际上使用的是 opencv-contrib-python，而非 Anaconda 中的 opencv，而前者是不带 H264 编解码器支持的．最终导致你命名安装了 Anaconda 的 opencv，但是依然出错．

建议先检查当前环境中的 opencv 相关的来自何处，如果 `conda list | grep opencv` 看到有来自 PyPI 的 `opencv-contrib-python`，只需将其卸载掉即可：

```bash
pip uninstall opencv-contrib-python -y
```

无意之中踩坑，挣扎许久，记录下来，希望对后来的人有用．
