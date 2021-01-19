---
title: "快速理解 numpy 中的数组拼接"
date: 2021-01-18T00:00:00+08:00
---

考虑我们有两个 `(5, 3)` 的数组，将他们拼接起来，输出数组的形状可以为 `(10, 3), (5, 6), (2, 5, 3), (5, 3, 2), (30, )`．可以看到，输出结果可能增加、减少轴的数量，或者保持不变．

实际上，Numpy 中用来拼接数组的函数就两个：`np.concatenate` 和 `np.stack`．其他的方法可以看作是语法糖，更加方便使用而已．一般地，`np.concatenate` 不改变轴的数量，除非你指定 `axis=None`；而 `np.stack` 会在你指定的 `axis` 上增加一个轴．具体来看看：

```python
import numpy as np

x = np.random.randn(5, 3)
y = np.random.randn(5, 3)

# z1.shape 为 (30, )
# 因为 axis=None 说明要将 x, y flatten 之后再拼接
z1 = np.concatenate([x, y], axis=None)
# z2.shape 为 (10, 3)
z2 = np.concatenate([x, y], axis=0)
# z3.shape 为 (5, 6)
z3 = np.concatenate([x, y], axis=-1)
# z4.shape 为 (2, 5, 3)
z4 = np.stack([x, y], axis=0)
# z5.shape 为 (5, 3, 2)
z5 = np.stack([x, y], axis=-1)
```

另外还有 `np.vstack, np.hstack, np.dstack` 分别在水平、垂直和深度三个方向上的拼接，实际上是 `np.concatenate` 分别设定 `axis=0, 1, 2`．不过注意到这里 `x` 和 `y` 是 2D 的数组，实际上 `axis=2` 是不能直接用的，需要增加一个轴进去才行．我们使用 `np.testing.assert_allclose` 来检查一下两个数组的值是否相等：

```python
from numpy.testing import assert_allclose

assert_allclose(np.hstack([x, y]), np.concatenate([x, y], axis=1))
assert_allclose(np.vstack([x, y]), np.concatenate([x, y], axis=0))
assert_allclose(np.dstack([x, y]), np.concatenate([np.expand_dims(x, -1), np.expand_dims(y, -1)], axis=2))
```

对于 2D 数组，numpy 中还有 `np.row_stack` 和 `np.column_stack`，当然也是语法糖了．他们分别等价于对 2D 数组使用 `np.concatenate` 拼接的之后指定 `axis=0, 1`．同样，我们用 `np.testing.assert_allclose` 来检查两者是否一致：

```python
assert_allclose(np.row_stack([x, y]), np.concatenate([x, y], axis=0))
assert_allclose(np.column_stack([x, y]), np.concatenate([x, y], axis=1))
```

## 小结

1. 当我们需要将数组沿着某个已有的轴拼接而不增加轴的数量时，使用 `np.concatenate`，具体到 2D 数组，我们还可以用 `np.row_stack, np.column_stack, np.hstack, np.vstack` 而不用指定 `axis` 参数，对 3D 数组，我们还有 `np.dstack` 可以使用．
2. 当我们需要将数组拼接并且增加一个新的轴时，我们就需要 `np.stack` 了，其中 `axis` 参数指定新轴的位置，一般都是放在最前（默认）或者最后．
3. 我们很少会 `np.concatenate([x, y], axis=None)`，因为这样实际上是 `np.concatenate([x.flatten(), y.flatten()], axis=0)`．
