---
title: "LamdaLR in PyTorch"
date: 2018-12-18T00:00:00+08:00
---

PyTorch 中学习率的调整，可以用 `torch.optim.lr_scheduler` 来做，官方已经提供了几个比较常用的 scheduler 了，比如按迭代次数衰减的 `StepLR`，更灵活的衰减迭代次数设置可以用 `MultiStepLR`，按指数衰减的 `ExponentialLR`，余弦退火的 `CosineAnnealingLR`，以及根据准确率等指标是否提升或者 `loss` 是否下降来确定学习率衰减的 `ReduceLROnPlateau`．

但是，这些都不是本文想要说的重点．除此之外，PyTorch 还提供了 `LambdaLR`，可以方便我们自定义 `lr_scheduler`．`LambdaLR` 的参数最重要的就是 `lr_lambda`，它可以是一个函数，或者一组函数．这个函数的输入参数为 `epoch`，也就是当前迭代次数，返回值是初始学习率要乘上的因子．然后，它就会将初始学习率乘上这个函数的返回值作为新的学习率．考虑到优化器里 `optimizer` 的参数 `param_group` 可以有多个，所以对应的，`LambdaLR` 的这个 `lr_scheduler` 也可以是多个函数．这个函数可以是我们自定义的函数，也可以是 `lambda` 表达式．

举例来说，假设我们希望学习率设置为：
$$
\mathit{lr} = \mathit{init\\_lr} * \Big(1 - \frac{\mathit{epoch}}{\mathit{num\\_epochs}}\Big)^{\mathit{power}}
$$
那么，我们定义这样的一个函数：

```python
def poly_lr_scheduler(epoch, num_epochs=300, power=0.9):
    return (1 - epoch/num_epochs)**power
```

然后，我们的 `lr_scheduler` 就是：

```python
lr_scheduler = LambdaLR(optimizer=optimizer, lr_lambda=poly_lr_scheduler)
```

我们可以写一段简短的代码来测试一下：

```python
from torch.optim import SGD
from torch.optim.lr_scheduler import LambdaLR
from torchvision.models import resnet18


def poly_scheduler(epoch, num_epochs=300, power=0.9):
    return (1 - epoch/num_epochs)**power


model = resnet18()
optimizer = SGD(model.parameters(), lr=0.001)
lr_scheduler = LambdaLR(optimizer=optimizer, lr_lambda=poly_lr_scheduler)


for epoch in range(300):
    lr_scheduler.step()
    print("learning rate: {:.6f}".format(optimizer.param_groups[0]['lr']))
    optimizer.zero_grad()
```

## 参考

* [torch.optim.lr_scheduler.LambdaLR](https://pytorch.org/docs/stable/optim.html#torch.optim.lr_scheduler.LambdaLR)