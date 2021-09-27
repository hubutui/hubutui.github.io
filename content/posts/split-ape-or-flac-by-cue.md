---
title: "根据 cue 分割无损音乐文件"
date: 2015-10-18T00:00:00+08:00
---

首先你要准备好一个无损音乐整轨文件, 一个 cue 文件. 其次, 工欲善其事，必先利其器, 准备好 shntool, cuetools. 举个例子, 林俊杰的学不会专辑 flac, 为了方便起见, 文件名叫设为 `lnf.flac`, 对应的 cue 文件为 `lnf.flac.cue`. 

一般来说我们找到的 cue 文件基本都是 GBK/GB2312 编码的, 先用 iconv 转换成 UTF-8 编码. 同时注意一下, 音轨文件 lnf.flac 应该是和 lnf.flac.cue 中的 FILE 项相对应的; 如果不对的话, 修改一下, 不然会找不到文件的. 

使用以下命令分割音轨: 

```bash
shntool split -f lnf.flac.cue -t '%n. %t' -o flac lnf.flac
```

其中 `shntool split` 与 `shnsplit` 等价; `-t` 选项设置输出文件的文件名格式, 详情参考 `shnsplit -h`; `-o` 设置文件类型, 具体支持的文件类型参考 `shntool -f`.

添加 tag 信息: 

```bash
cuetag lnf.flac.cue [split flac files]
```

有的发行版是用 `cuetag.sh`, 至于 ape 格式的, 可以先用 ffmpeg 转换成 flac 格式之后再操作: 

```bash
ffmpeg -i lnf.ape -acodec flac lnf.flac
```

* 参考 [Split .ape files with cue and convert to flac](https://blogs.gnome.org/happyaron/2009/08/01/split-ape-cue-convert-flac/)
