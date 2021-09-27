---
title: "pandas 基础数据结构简介"
date: 2021-01-16T20:00:00+08:00
---

## 前言

本文对 pandas 中的基础数据结构 `Series` 和 `DataFrame` 进行简洁的介绍．

## 正文

首先，我们考虑如何表达数据．最简单的方式就是用一个表格来表示．例如，学生成绩表可以用如下表格表示：

| 姓名 | 语文 | 数学 | 英语 |
| ---- | ---- | ---- | ---- |
| 张三 | 89   | 87   | 67   |
| 李四 | 87   | 88   | 64   |

我们进一步把这个表格拆分为不同的列，即有姓名、语文、数学和英语四列．这样，在 pandas 中我们就可以用 `Series` 来表示每一列．也就是说，`Series` 实际上就是一维的数组．也就是：

```python
import pandas as pd
import numpy as np
name = pd.Series(["张三", "李四"], name="姓名")
yuwen = pd.Series([89, 87], name="语文")
math = pd.Series([87, 88], name="数学")
english = pd.Series([67, 64], name="英语")
```

我们把 `Series` 拼接起来，就是构成了 `DataFrames`．

```python
df = pd.DataFrame({name.name: name, yuwen.name: yuwen, math.name: math, english.name: english})
print(df)
```

输出结果：

```text
   姓名  语文  数学  英语
0  张三  89  87  67
1  李四  87  88  64
```

下一个问题就是，我们如何选取 `DataFrame` 中不同列和不同行．可以看到，`DataFrame` 有两个重要的属性，`index` 和 `columns`，前者索引的是不同的行，后者索引的是不同的列．例如，我们想要查看所有学生的语文成绩：

```python
print(df['语文'])
```

## 小结

本文对 pandas 中的基础数据结构 `Series` 和 `DataFrame` 做了简单的介绍．更多的时候，我们是在对 `DataFrame` 这种数据类型上操作，时刻记住 `index` 和 `columns` 分别指二维数组（表格）中的行和列，是非常重要的．更加详细的信息，请参考文后的参考文献．

## 参考

1. https://pandas.pydata.org/docs/user_guide/dsintro.html
