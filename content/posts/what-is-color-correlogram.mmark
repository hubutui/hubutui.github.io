---
title: "什么是颜色相关直方图"
date: 2017-12-07T00:00:00+08:00
markup: mmark
---

## 正文
记 $$ I $$ 为一幅 $$ n \times n $$ 的图像，$$ I $$ 中的颜色可以量化为 $$ m $$ 种：$$ c_{1}, c_{2}, \ldots, c_{m} $$．对于一个像素 $$ p = (x, y) $$，记 $$ I(p) $$ 为他的颜色．记 $$ I_{c} \triangleq \{ p \mid I(p) = c \} $$，这样子的话，记号 $$ p \in I_{c} $$ 就等价于 $$ p \in I, I(p) = c $$．为了简单起见，作者使用无穷范数来度量两个像素之间的距离，即像素 $$ p_{1} = (x_{1}, y_{1}), p_{2} = (x_{2}, y_{2}) $$ 的距离为 $$ \lvert p_{1} - p_{2} \rvert \triangleq \max \{ \lvert x_{1}, x_{2} \rvert, \lvert y_{1} - y_{2} \rvert \} $$．记集合 $$ \{ 1, 2, \ldots, n \} $$ 为 $$ [n] $$．

图像 $$ I $$ 的直方图 $$ h $$ 定义为：

$$
h_{c_{i}}(I) \triangleq n^{2} \cdot \Pr_{p \in I} [p \in I_{c_{i}}], \quad i \in [m].
$$

这里的 $$ \Pr $$ 是概率 Probability 的意思．易知，对图像中的任意像素，$$ h_{c_{i}} / n^{2} $$ 就给出了他的颜色为 $$ c_{i} $$ 的概率．

令距离 $$ d \in [k] $$ 为固定值，则 $$ i, j \in [m], k \in [d] $$ 时的图像 $$ I $$ 的颜色相关图定义为：

$$
\gamma_{c_{i}, c_{j}}^{(k)} (I) \triangleq \Pr_{p_{1} \in I_{c_{i}}, p_{2} \in I} [p_{2} \in I_{c_{j}} \mid \lvert p_{1} - p_{2} \rvert = k].
$$

## 后续

这篇文章是之前读这篇[论文](http://ieeexplore.ieee.org/abstract/document/7937848/)的时候遇到颜色直方图，不知道颜色直方图到底是什么，然后根据各种搜索结果整理而成的．后来看看，实际上那篇论文并没有太多的涉及，毕竟人家提出的是改进的颜色直方图嘛．

## 参考

* [Combining Color and Spatial Information for Content-based Image Retrieval](http://www.cs.cornell.edu/rdz/Papers/ecdl2/spatial.htm)
* [颜色相关直方图](http://blog.sina.com.cn/s/blog_610f088301012kjh.html)
