---
title: "在图像上叠加绘制分割结果的方法"
date: 2021-11-28T08:00:00+08:00
---

## 简介

在进行图像分割之后，我们常常需要将分割的结果叠加绘制到原图上，并以不同的颜色显示，以便进行展示．

## 已有的方案

scikit-image，opencv，gluoncv, detectron2 中都提供有类似功能的函数．不过还是存在一定的缺点：

1. scikit-image 中的 [skimage.color.label2rgb](https://scikit-image.org/docs/dev/api/skimage.color.html#skimage.color.label2rgb) 只能处理灰度图像，彩色图像也是必须转换为灰度图像之后才将分割结果绘制上去．只能说勉强能用．
2. opencv 中的 [cv2.addWeighted](https://docs.opencv.org/4.5.4/d2/de8/group__core__array.html#gafafb2513349db3bcff51f54ee5592a19) 实际上是用于将两个图像按照不同的比例进行融合．将其简单的用于图像与分割结果的融合，会发现在没有分割目标的区域灰度降低，即整体的图像变暗了．其实不算是很好的解决方案．
3. detectron2 中的 [detectron2.utils.visualizer.Visualizer](https://detectron2.readthedocs.io/en/latest/modules/utils.html#detectron2.utils.visualizer.Visualizer) 也提供了类似的功能，但是文档说其为了渲染质量而牺牲了速度，不易在需要实时展示的场景使用．而且为了找个功能而单独安装 detectron2 有点得不偿失．但是这部分代码却很难单独抽取出来使用．
4. gluoncv 提供的 [gluoncv.utils.viz.plot_mask](https://cv.gluon.ai/api/utils.html#gluoncv.utils.viz.plot_mask) 代码简洁．我们可以将其单独提取出来，并且简单修改一下，更加好用．

## 修改后的版本

基于 `gluoncv.utils.viz.plot_mask`，我将其简单的修改，代码如下：

```python
import numpy as np

def plot_mask(img, masks, colors=None, alpha=0.5) -> np.ndarray:
    """Visualize segmentation mask.

    Parameters
    ----------
    img: numpy.ndarray
        Image with shape `(H, W, 3)`.
    masks: numpy.ndarray
        Binary images with shape `(N, H, W)`.
    colors: numpy.ndarray
        corlor for mask, shape `(N, 3)`.
        if None, generate random color for mask
    alpha: float, optional, default 0.5
        Transparency of plotted mask

    Returns
    -------
    numpy.ndarray
        The image plotted with segmentation masks, shape `(H, W, 3)`

    """
    if colors is None:
        colors = np.random.random((masks.shape[0], 3)) * 255
    else:
        if colors.shape[0] < masks.shape[0]:
            raise RuntimeError(
                f"colors count: {colors.shape[0]} is less than masks count: {masks.shape[0]}"
            )
    for mask, color in zip(masks, colors):
        mask = np.stack([mask, mask, mask], -1)
        img = np.where(mask, img * (1 - alpha) + color * alpha, img)

    return img.astype(np.uint8)
```

由于最近正好在看如何用 opencv C++ 写代码，我还写了一个简单的 opencv C++ 版本：

```c++
#include <opencv2/opencv.hpp>
using namespace cv;
void plot_mask(Mat image, Mat mask, float alpha = 0.5, Scalar color = Scalar(0, 255, 0))
{
    auto img_iter = image.begin<Vec3b>();
    auto mask_iter = mask.begin<uchar>();

    while (img_iter != image.end<Vec3b>())
    {
        if (*mask_iter)
        {
            (*img_iter)[0] = (*img_iter)[0] * (1 - alpha) + color[0] * alpha;
            (*img_iter)[1] = (*img_iter)[1] * (1 - alpha) + color[1] * alpha;
            (*img_iter)[2] = (*img_iter)[2] * (1 - alpha) + color[2] * alpha;
        }
        img_iter++;
        mask_iter++;
    }
}
```

不过很明显，这个 C++ 版本的函数只能处理 `mask` 中只有一类标签的，有多个类别的可以多次调用该函数处理．相信这个修改并不算太难，只是目前暂时先不更新了．