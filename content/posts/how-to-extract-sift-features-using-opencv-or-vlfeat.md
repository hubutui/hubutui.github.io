---
title: "如何使用 opencv 或者 vlfeat 提取 sift 特征"
date: 2020-06-04T00:00:00+08:00
---

本文简略介绍一下如何提取 sift 特征的方法，平台为 Linux，编程语言为 python。当然，实际上 Windows 也可以参考一下。

sift 特征的介绍请自行百度，这里不涉及。我们假定用户已然知晓 sift 特征是什么，我们只考虑如何用 python 编程去提取 sift 特征。

可以用于提取 sift 特征的有两个库，第一个是大名鼎鼎的 opencv，但是 sift 特征提取的实现放在 opencv-contrib。另外一个是 vlfeat，不过 [vlfeat](https://www.vlfeat.org/) 是 C 库，且只提供了 MATLAB 接口，不过好在 [cyvlfeat](https://github.com/menpo/cyvlfeat) 提供了相应的 python 接口。

## 安装

### opencv

opencv 的安装比较简单，可以直接用你的发行版提供的包管理器直接安装。不过可能部分发行版没有把 opencv-contrib 编译打包。另外一个比较简单的方法就是使用 Anaconda 提供的 opencv，请确保从 anaconda channel 而非 conda-forge 安装：

```shell
conda install -c anaconda opencv
```

### vlfeat & cyvlfeat

ArchLinux 用户可以直接用这个 [PKGBUILD](https://aur.archlinux.org/cgit/aur.git/tree/PKGBUILD?h=vlfeat) 打包一个 vlfeat，其他发行版用户可以参考该文件中的 `build` 函数的内容进行编译，参考其中的 `package` 函数的内容进行安装。安装完毕的 vlfeat 包应该是类似如下的目录结构：

```text
vlfeat /usr/bin/aib
vlfeat /usr/bin/mser
vlfeat /usr/bin/sift
vlfeat /usr/include/vl/
vlfeat /usr/include/vl/aib.h
vlfeat /usr/include/vl/array.h
vlfeat /usr/include/vl/covdet.h
vlfeat /usr/include/vl/dsift.h
vlfeat /usr/include/vl/fisher.h
vlfeat /usr/include/vl/generic.h
vlfeat /usr/include/vl/getopt_long.h
vlfeat /usr/include/vl/gmm.h
vlfeat /usr/include/vl/heap-def.h
vlfeat /usr/include/vl/hikmeans.h
vlfeat /usr/include/vl/hog.h
vlfeat /usr/include/vl/homkermap.h
vlfeat /usr/include/vl/host.h
vlfeat /usr/include/vl/ikmeans.h
vlfeat /usr/include/vl/imopv.h
vlfeat /usr/include/vl/imopv_sse2.h
vlfeat /usr/include/vl/kdtree.h
vlfeat /usr/include/vl/kmeans.h
vlfeat /usr/include/vl/lbp.h
vlfeat /usr/include/vl/liop.h
vlfeat /usr/include/vl/mathop.h
vlfeat /usr/include/vl/mathop_avx.h
vlfeat /usr/include/vl/mathop_sse2.h
vlfeat /usr/include/vl/mser.h
vlfeat /usr/include/vl/pgm.h
vlfeat /usr/include/vl/qsort-def.h
vlfeat /usr/include/vl/quickshift.h
vlfeat /usr/include/vl/random.h
vlfeat /usr/include/vl/rodrigues.h
vlfeat /usr/include/vl/scalespace.h
vlfeat /usr/include/vl/shuffle-def.h
vlfeat /usr/include/vl/sift.h
vlfeat /usr/include/vl/slic.h
vlfeat /usr/include/vl/stringop.h
vlfeat /usr/include/vl/svm.h
vlfeat /usr/include/vl/svmdataset.h
vlfeat /usr/include/vl/vlad.h
vlfeat /usr/lib/libvl.so
```

简单地说，就是把 `aib`，`mser` 和 `sift` 三个二进制文件安装到了 `/usr/bin` 目录下，所有的头文件全部安装到 `/usr/include/vl` 目录下（这些头文件在下一步安装 cyvlfeat 的时候会用到），动态链接库 `libvl.so` 安装到 `/usr/lib`。其他发行版可能使用不同的目录结构，请根据需要修改。

然后安装 cyvlfeat，ArchLinux 用户可以直接用这个 [PKGBUILD](https://aur.archlinux.org/cgit/aur.git/tree/PKGBUILD?h=python-cyvlfeat) 编译打包 cyvlfeat，然后安装。其他发行版的用户也可以参考该文件自行编译和安装。

此外，还有一种更加简单的安装方式，直接用 Anaconda：

```shell
conda install -c mepo cyvlfeat
```

vlfeat 会作为 cyvlfeat 的依赖项自动安装到 conda 环境中。

## 提取 sift 特征的代码

### opencv

我们首先用 opencv 来提取特征，并测试一下速度：

```python
import cv2
from cv2.xfeatures2d import SIFT_create

filename = 'cameraman.tiff'
img = cv2.imread(filename)
sift_detector = SIFT_create()
%timeit kps, descriptors_cv2 = sift_detector.detectAndCompute(img, None)
```

结果如下：

```text
17.9 ms ± 954 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```

看起来速度还不错（不同的机器配置结果会有所不同）。我们把提取到的 sift 特征点以及它的大小方向都可视化出来看一下：

```python
import cv2
from cv2.xfeatures2d import SIFT_create
import matplotlib.pyplot as plt

filename = 'cameraman.tif'
img = cv2.imread(filename)
sift_detector = SIFT_create()
kps, descriptors = sift_detector.detectAndCompute(img, None)
cv2.drawKeypoints(img, kps, outImage=img, flags=cv2.DRAW_MATCHES_FLAGS_DRAW_RICH_KEYPOINTS)
plt.imshow(img)
plt.axis('off')
plt.show()
```

![opencv-sift](/img/opencv-sift-viz.png)

### vlfeat && cyvlfeat

再来试试用 vlfeat 来提取特征，并测试一下速度：

```python
import cv2
import numpy as np
from cyvlfeat.sift import sift

filename = 'cameraman.tif'
img = cv2.imread(filename, flags=cv2.IMREAD_GRAYSCALE)
img = img.astype(np.float32)
%timeit frames, descriptors = sift(img, compute_descriptor=True)
```

结果如下：

```text
32.1 ms ± 272 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
```

明显 vlfeat 的速度是不如 opencv 的。建议如果可以的话，优先使用 opencv。opencv 和 vlfeat 还有一些区别：

1. vlfeat 只能处理灰度图，而 opencv 可以处理彩色图，不过可能是在处理的时候讲彩色图转换为了灰度图。
2. vlfeat 要求输入图像的 numpy 数组的数据类型为 `np.float32`，默认返回的描述子的数据类型却是 `np.uint8`，当然你可以指定参数 `float_descriptors=True`。
3. vlfeat 还可以计算 dense sift 特征，而 opencv 从某个版本开始把 dsift 特征提取的功能给删掉了。当然，实际上也有其他的一些解决方法，只是写起来麻烦了点，请自行谷歌吧。
4. opencv 可以提供 surf 特征，而 vlfeat 没有此功能。当然，vlfeat 也可以提取其他 opencv 没有提供的特征，但是这些方法对应的 python 接口却是 cyvlfeat 没有提供的。cyvlfeat 已经很久没有更新了，如果有需要的话，可以看看它的 [pr 列表](https://github.com/menpo/cyvlfeat/pulls)，添加自己需要的功能。

我们把 vlfeat 提取的 sift 特征也可视化一下看看：

```python
import cv2
import numpy as np
from cyvlfeat.sift import sift
import matplotlib.pyplot as plt

filename = 'cameraman.tif'
img = cv2.imread(filename)
frames, descriptors = sift(cv2.cvtColor(img, cv2.COLOR_BGR2GRAY).astype(np.float32), compute_descriptor=True)
kps = [cv2.KeyPoint(*frame.tolist()) for frame in frames]
cv2.drawKeypoints(img, kps, outImage=img, flags=cv2.DRAW_MATCHES_FLAGS_DRAW_RICH_KEYPOINTS)
plt.imshow(img)
plt.axis('off')
plt.show()
```

![vlfeat-sift](/img/vlfeat-sift-viz.png)

可以看到，opencv 与 vlfeat 提取到的特征点的数量不同，具体的特征点也有所不同，最后提取到的 sift 特征也有区别。这主要是两者在实现上的差异。具体的细节可能需要去阅读他们的源码才能了解了。

## 更新（20200611）

最近给 cyvlfeat 做了一点微小的贡献，把一个贡献者提交的代码进行了一些修改和补充，把 vlfeat 中的 vlad, phow, flatmap, quickshift 等函数的 Python 绑定都提供出来了。感觉应该需要把 kdtree 的接口也提供一下的，毕竟 vlad 应该是用得上这个的。但是 cython 真的挺不好写啊，或者还是自己水平有限，无法理解。也许可以用 scikit-learn 中的函数替代，有空可以看看。

## 参考

1. https://www.vlfeat.org/overview/sift.html
2. https://yongyuan.name/blog/extract-sift-and-surf-descriptor-using-opencv.html
3. https://docs.opencv.org/4.3.0/d5/d3c/classcv_1_1xfeatures2d_1_1SIFT.html