---
title: "利用 SimpleITK 对医学图像进行重采样"
date: 2021-08-24T00:00:00+08:00 
---

在医学图像处理中，我们的算法通常会默认图像中的像素/体素是各向同性的，也就是对于 2D 图像来说，他在 X 和 Y 方向上每个像素之间距离是相同的，不能说 X 轴方向上的 1 像素对应到实际的物理尺寸为 1mm，而 y 轴方向为 2mm；对于 3D 图像，我们还要求 Z 轴方向也是如此．因此，常常需要在将这些图像进行简单的重采样处理，以保证他们是各向同性的．

通常，医学图像数据可以使用 SimpleITK 来读写和处理，这里，我们采用 SimpleITK 中的 Resample 来进行医学图像的重采样．废话不多说，上代码：

```python
import numpy as np
import SimpleITK as sitk

def resample_img(src_img, interpolator=sitk.sitkLinear, dest_spacing=[1., 1., 1.]):
    src_size = src_img.GetSize()
    src_spacing = src_img.GetSpacing()
    dest_size = np.array(src_size) * np.array(src_spacing) / np.array(dest_spacing)
    dest_size = np.round(dest_size).astype(np.int64).tolist()

    return sitk.Resample(src_img, dest_size, sitk.Transform(), interpolator,
                         src_img.GetOrigin(), dest_spacing, src_img.GetDirection(),
                         0, src_img.GetPixelID())
```

使用上述函数，即可方便地对医学图像进行重采样处理．该函数接收三个参数，第一个参数是输入图像，第二个参数为插值方法，默认采用线性插值，这是一种比较简单的插值方式，同时兼顾了性能和速度，第三个参数是输出图像的像素间隔（pixel spacing），一般地我们设置其默认值为 `[1., 1., 1.]` 即可．

更多信息，请参考以下文档：

1. [sitk.Resample](https://simpleitk.org/doxygen/latest/html/namespaceitk_1_1simple.html#a785a433b01484e8390f1f2723807cf65)

2. [Fundamental Concepts](https://simpleitk.readthedocs.io/en/master/fundamentalConcepts.html)
