---
title: "医学图像中的窗口化处理"
date: 2020-12-08T00:00:00+08:00
---

## 基础概念

在计算机中，我们常见的图像的灰度值取值为 0 到 255，一般用 8 个比特来表示．相应的，我们的显示器也是适合显示 256 灰度级的图像．

然而，医学图像的灰度级范围取值往往更大，超过了 255．这时候想要在显示器上正确显示的话，我们需要进行一个线性映射．这种操作通常称为窗口化（windowing）．

实际上，在 DICOM 标准中，有窗位（Window Center，也称 Window Level）和窗宽（Window Width）的 DICOM tag 定义．一些软件，如 ITK-SNAP，3D Slicer 均会自动读取 DICOM 数据中定义的窗位和窗宽，然后进行窗口化操作，以正常地显示图像．

## 窗口化

![image-20201208130532883](/img/mri-windowing-itk-snap.png)

如图所示，这是一个 MRI 图像的窗位窗宽设定．记窗位为 $y_\mathrm{c}$，窗宽为 $y_\mathrm{w}$，则我们可以计算得到窗口中的最大和最小灰度值分别为：
$$
y_\mathrm{min} = y_\mathrm{c} - 0.5 \times y_\mathrm{w} \\
y_\mathrm{max} = y_\mathrm{c} + 0.5 \times y_\mathrm{w}
$$
记输入图像的灰度值为 $s$，输出图像的灰度值为 $r$，则灰度线性变换可以用下式定义：
$$
r = a \times s + b
$$
对应到窗口化，这实际上是是一个分段线性变换：
$$
r = \begin{cases} 
	0, & s < y_\mathrm{min} \newline
	\dfrac{1}{y_\mathrm{w}}s + y_\mathrm{min}, & y_\mathrm{min} \leq s \leq y_\mathrm{max} \newline
	1, & s > y_\mathrm{max}
\end{cases}
$$

## 具体实现

SimpleITK 是医学图像处理中比较常用的工具包，我们可以简单的使用其来实现 Windowing 操作．大致的代码如下：

```python
import SimpleITK as sitk

# 假设我们有一个 DICOM Series，存放在 example-data 目录下
dicomdir = 'example-data'
reader = sitk.ImageSeriesReader()
reader.SetFileNames(reader.GetGDCMSeriesFileNames(dicomdir, recursive=True))
# 我们需要从 MetaData 中读取 window center 和 window width
reader.MetaDataDictionaryArrayUpdateOn()
window_center_tag = '0028|1050'
window_width_tag = '0028|1051'
img = reader.Execute()
# 执行了 Execute() 之后才能读取 MetaData
# 返回结果是 str，转换为 int
window_center = int(reader.GetMetaData(slice_idx, window_center_tag))
window_width = int(reader.GetMetaData(slice_idx, window_width_tag))
# 计算窗口范围内的最大和最小灰度值
window_min = window_center - window_width * 2
window_max = window_center + window_width * 2
# 直接调用 sitk.IntensityWindowing filter 来进行窗口化操作
img_windowing = sitk.IntensityWindowing(img, window_min, window_max)
```

