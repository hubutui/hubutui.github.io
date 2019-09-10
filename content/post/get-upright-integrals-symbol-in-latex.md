---
title: "LaTeX 中的直立积分符号"
date: 2016-07-18T00:00:00+08:00
---

貌似 LaTeX 中各种字体宏包的积分符号都是倾斜的，与大陆教科书上常用的直立的积分符号不太一样．不过，既然这么多年看习惯了直立的积分符号，到了 LaTeX 下自然也想用直立的积分符号了． 当然，这有很多种方法． 

首先是我最推荐的方法，使用 `unicode-math` 宏包，并将字体设置为 XITS Math 字体． 

```tex
\usepackage{unicode-math}
\setmathfont[StylisticSet = 8]{xits-math.otf}
```

这里 `StylisticSet = 8` 就是为了使用直立的积分符号．关于 `unicode-math` 的用法，请参考其手册，直达命令 `texdoc unicode-math`．关于 XITS Math 字体，可以参考维基百科 [XITS Math 字体](https://en.wikipedia.org/wiki/XITS_font_project)，或者它的手册，直达命令 `texdoc xits`．关于 `StylisticSet = 8` 是什么鬼东西，你需要参阅 `fontspec` 宏包的手册中的 Stylistic Set variations 一节的内容，直达命令 `texdoc fontspec`． 

其次，你也可以用 `stix` 宏包，它提供了一个 `upint` 选项，可以得到直立的积分符号．它还专门为直立的积分符号定义了命令来方便使用．详情参考它的手册，直达命令 `texdoc stix`． 

再次，你还可以直接使用特定字体，这些字体的积分符号就是直立的．比如 Asana Math，Neo Euler，以及 M$ Office 附带的 Cambria Math 字体． 

最后，你甚至可以自己将倾斜的积分符号经过旋转一定的角度来得到直立的积分符号，不过这个方法也比较复杂，还需要注意调整上下限的位置等等，不推荐使用．
