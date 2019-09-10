---
title: "使用 SimpleScreenRecorder 录制桌面 - 迄今为止 Linux 下最佳的解决方案"
date: 2015-09-27T00:00:00+08:00
---

提到 Linux 下的屏幕录制软件, recorditnow 已经很多年不更新了, 安装完还不一定能用; Record my desktop 比 recorditnow 还更糟糕; Kazam 没有什么印象, 但是谷歌搜索连个项目主页都没有, 上游 URL 在 [launchpad](https://launchpad.net/kazam), 恐怕也是不成气候; vokoscreen 听说还不错, 但是没有详细了解. 但是今天要说的主角是 SimpleScreenRecorder. 能与之竞争的也就 vokoscreen 了. 当然你要是直接使用 ffmpeg 录制就当我什么都没说吧.  

## 简介

还是废话几句吧. SimpleScreenRecorder 是 Linux 下的一个桌面录制软件, 简单易用, 实乃居家旅行~~杀人越货~~必备佳品啊. 看看这些特性: 

  * Graphical user interface (Qt-based). 这就不用说了, 友好的 Qt4 界面.
  * Faster than VLC and ffmpeg/avconv. 秒杀 VLC, 拳打 ffmpeg.
  * Records the entire screen or part of it, or records OpenGL applications directly (similar to Fraps on Windows). 多么人性化, 想要怎样录制就怎样.
  * Reduces the video frame rate if your computer is too slow (rather than using up all your RAM like VLC does). 对于老旧机器, 选择降低码率而非多吃点内存, 有效避免了系统假死导致心情不悦最后怒摔电脑的结局.
  * Fully multithreaded: small delays in any of the components will never block the other components, resulting is smoother video and better performance on computers with multiple processors. 允许跳帧, 保证最终播放效果如丝班顺滑.
  * Pause and resume recording at any time (either by clicking a button or by pressing a hotkey). 可以随时暂停和恢复录制, 可以直接使用快捷键, 避免录制到 SimpleScreenRecorder; 要我说这么好的东西应该让大家都知道才对啊.
  * Shows statistics during recording (file size, bit rate, total recording time, actual frame rate, ...). 实时了解录制详情, 文件大小, 比特率, 已录制时间, 实际帧率等等.
  * Can show a preview during recording, so you don't waste time recording something only to figure out afterwards that some setting was wrong. 还能查看预览, 最终效果如何先睹为快, 免得最后浪费时间录制完成结果效果不符合自己的预期.
  * Uses libav/ffmpeg libraries for encoding, so it supports many different codecs and file formats (adding more is trivial). 使用 ffmpeg 编码, 只要 ffmpeg 支持, 想用哪种编解码器哪种容器都可以的嘛.
  * Can also do live streaming (experimental). 还能做视频直播, 比如一边玩 Minecraft 一边直播到 twitch.tv.
  * Sensible default settings: no need to change anything if you don't want to. 以及精心调教好的预置配置, 懒人必备.
  * Tooltips for almost everything: no need to read the documentation to find out what something does. 还有更多特性请求等你来提. 

## 安装

废话已经很多了, 不能说了. 先安装吧, Arch Linux 用户可以直接安装 AUR 里的 [simplescreenrecorder-git](https://aur.archlinux.org/packages/simplescreenrecorder-git/), Arch Linux 官方源提供的那个 simplescreenrecorder 因为错误依赖了 Qt5 导致无法使用, 现已修复．

## 如何使用

安装完直接打开基本上就可以用了. 推荐观看这个[视频教程](https://www.youtube.com/watch?v=PC68a2IT6qA), 虽然是英文的, 但是内容比较简单, 可以打开 Youtube 自动字幕, 如果还有困难还可以打开翻译. 关于后端选择, 虽然 simplescreenrecorder 推荐使用 jack, 但是我更加推荐 pulseaudio, 基本不会出什么错误, 也不需要复杂的配置. 需要注意的一点是, 如果要同时录制麦克风声音和电脑上的声音的话, 需要: 

```bash
pactl load-module module-loopback
```

这一点上文的视频教程也有提到. 或者参考[这里](http://www.maartenbaert.be/simplescreenrecorder/recording-game-audio/#the-easy-way).
