---
title: "使用 lxml 快速对 XML 文件进行数据提取和清洗"
date: 2020-12-27T00:00:00+08:00
---

本文不会给出详细的代码细节，只是简单介绍一下思路，以及可能用到的库等。

## 什么是 XML

XML，即可扩展标记语言，是一种标记语言。

XML 自然而然的可以用树来表示，树的每一个节点就是对应着每一个元素。在 Python 中，我们可以用嵌套的字典来表示一棵树。实际上，python 中的 `xmltodict` 就可以直接将 XML 转换为字典。不过转换为字典之后，其实也还不够方便我们去查找内容。这时候就要介绍一下一个比较常用的 XML 解析与处理的库了：lxml。

## lxml

lxml 是一个功能丰富的 python 库，用于解析 XML 和 HTML 文件。它实际上是 libxml2 和 libxlst 这两个 C 语言库的 python 绑定，所以在速度上是非常不错的。

### 概念

要使用 lxml，我们首先得了解两个基础概念：Element 和 ElementTree。Element，实际上就是 XML 中元素的实现。Element 用字典来保存属性，用 `text` 成员来保存文本。很多个 Element 按照树的结构组织，就构成了 ElementTree。

### 使用 XPath 从 XML 中查找元素

XPath 即 XML 路径语言，专门用来在 XML 中查找节点（或者元素）的一种语言。其语法请参考[这里](https://www.w3schools.com/xml/xpath_syntax.asp)。在 lxml 中，Element 和 ElementTree 对象都可以用 XPath 表达式去查找，但是两者的支持略有不同。比如在 Element 对象中的 XPath 不能用绝对路径，也不支持使用 XPath Axes。但是，我们指导，如果我们找到了树中的一个节点，那么我们可以以这个节点作为根，构建得到一棵子树。所以，理论上 Element 和 ElementTree 的关系很紧密，而且应该可以转换的嘛。

事实上，假设我们用 XPath 表达式从 ElementTree 中找到了某个节点，返回的结果类型为 Element，然后我们想用另一个 XPath 表达式在这个节点及其子节点去查找某个节点。那么，为了用上完整的 XPath 支持，我们可以使用 `lxml.etree.ElementTree(element)` 去构建以 `element` 为根的子树，然后又可以愉快地使用 XPath 表达式去查找了。PS：感觉不知不觉中用上了递归。

## 参考

* [XPath 语法](https://www.w3schools.com/xml/xpath_syntax.asp)
* 维基百科上的 [XML 词条](https://zh.wikipedia.org/wiki/XML)
* lxml 的 [lxml.etree tutorial](https://lxml.de/tutorial.html)，了解 `lxml.etree` 就可以快速了解如何使用 lxml 了。
* lxml 的 [XPath 文档](https://lxml.de/xpathxslt.html) 也值得一看，毕竟有了一些扩展。
