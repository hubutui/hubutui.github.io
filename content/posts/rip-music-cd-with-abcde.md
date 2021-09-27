---
title: "Linux 下用 abcde 抓取音乐 CD 为无损音频"
date: 2015-10-25T00:00:00+08:00
---

我是真的没想到那么简单啊. 一条命令搞定诶. 首先你要有一张音乐 CD; 然后你需要安装 abcde: 

```bash
pacman -S abcde
```

然后使用以下命令抓轨: 

```bash
abcde -o flac -d /dev/cdrom
```

其中 `-o` 选项设置输出格式, abcde 支持 vorbis,mp3,flac,spx,mpc,wav,m4a,opus,wv,ape,mp2,tta 等格式. 这里我用了开源的无损格式 flac. 如果你没有安装 flac 的话: 

```bash
pacman -S flac
```

abcde 还提供了其他有用的选项, 可以自行查看帮助: 

```bash
abcde -h
```

考虑到某些玄学说抓取结果可能不准确, 可以多次抓取然后对比 checksum. 最好还使用全新的音乐 CD. 抓取结束之后就可以将 CD 收藏起来啦. PS: abcde 是 A Better CD Encoder 的首字母缩写, 很有意思的一个名字, 主页在[这里](http://abcde.einval.com/wiki/).
