---
title: "关于病理图像分析常用的几个库以及一些踩坑记录"
date: 2024-03-06T00:00:00+08:00
toc: false
images:
tags:
  - bigtiff
  - libvips
  - openslide
  - pyramid
  - pyvips
  - rasterio
  - tiled tiff
  - 图像金字塔
  - 病理图像
---

## TIFF

标签图像文件格式（Tagged Image File Format，简写为TIFF）是一种灵活的位图格式，是病理图像处理中常用的图像文件格式．

病理图像的尺寸通常很大，例如我们常用的示例图像[CMU-1.tiff](https://openslide.cs.cmu.edu/download/openslide-testdata/Generic-TIFF/CMU-1.tiff)的尺寸为46000x32914．在普通的计算机上直接去处理这样尺寸的图像是不现实的，也是很不方便的．

一般我们使用图像平铺和缩放金字塔技术来解决这个问题．它可以让我们用最小的内存和资源去处理任意尺度下的任意区域的图像数据．这里的平铺指的是 tiling，也就是将图像按照一定的 tile size（常用 256x256）进行分块，每一块就叫做一个 tile．图像编码和解码可以对 tile 进行操作，因此我们就可以根据需要只取读写所需的 tile，从而避免将整个图像全部读入内存而导致的 OOM．这里的 tile 所使用的图像编码技术可以根据需要选择，tiff 理论上应该支持各种编码，但是受限于具体的图像处理库的支持，一般会使用 LZW 或者 JPEG 编码．一般推荐使用 LZW 编码即可．

而缩放金字塔则是在保存图像原始尺寸的基础上，再将其下采样到不同的尺寸，并且保存在同一个文件中，程序处理的时候可以根据不同的需要在不同的尺度下进行处理．一般会选择 2 的幂次作为缩放的倍率，例如分别下采样到原图尺寸的 2、4、8、16倍，或者准确的表述，应该叫做下采样为原图的 1/2，1/4，1/8和1/16．

TIFF 文件并不要求这个图像必须是 tiled 的，也不要求这个图像必须是金字塔（pyramid）结构的．而病理图像中常用的库 openslide，要求这个 tiff 必需是 tiled 的，否则将会报错说这个文件格式不支持．这也就是为什么我们拿到的明明是 tiff 文件，但是 openslide 却报错无法处理．

## 如何生成和识别 tiled pyramid tiff

那么，对于需要使用 openslide 处理的场景，一个比较重要的问题就是如何生成 tiled tiff 了．

```python
from skimage import data
from PIL import Image

img = Image.fromarray(data.astronaut())
img.save("astronaut.tif")
img.save("astronaut.png")
```

我们来看一个例子，我们使用上述代码保存了两个文件 "astronaut.tif" 和 "astronaut.png"．这里的 "astronaut.tif" 是一个 tiff 文件，但是不是 tiled tiff．使用 `openslide-show-properties` 命令查看文件信息，结果如下：

```bash
openslide-show-properties astronaut.tif
openslide-show-properties: astronaut.tif: Not a file that OpenSlide can recognize
```

我们也可以使用 tiffinfo 命令查看一下这个 tiff 文件的信息，如下：

```bash
tiffinfo astronaut.tif
=== TIFF directory 0 ===
TIFF Directory at offset 0x8 (8)
  Image Width: 512 Image Length: 512
  Bits/Sample: 8
  Compression Scheme: None
  Photometric Interpretation: RGB color
  Samples/Pixel: 3
  Rows/Strip: 512
  Planar Configuration: single image plane
```

我们试试使用 imagemagick 来将 "astronaut.png" 转换为 tiff 文件，看看结果如何：

```bash
convert astronaut.png astronaut.1.tiff
openslide-show-properties astronaut.1.tif
openslide-show-properties: astronaut.1.tif: Not a file that OpenSlide can recognize
=== TIFF directory 0 ===
TIFF Directory at offset 0xc0008 (786440)
  Image Width: 512 Image Length: 512
  Bits/Sample: 8
  Compression Scheme: None
  Photometric Interpretation: RGB color
  FillOrder: msb-to-lsb
  Orientation: row 0 top, col 0 lhs
  Samples/Pixel: 3
  Rows/Strip: 512
  Planar Configuration: single image plane
  Page Number: 0-1
  White Point: 0.3127-0.329
  PrimaryChromaticities: 0.640000,0.330000,0.300000,0.600000,0.150000,0.060000
```

可以看到，如果没有指定额外的选项参数，imagemagick 转换得到的 tiff 也不是 tiled tiff．

事实上，我们可以修改一下 imagemagick 的转换命令选项参数，生成 tiled tiff，命令如下：

```bash
# 这里我们一般使用 lzw 压缩，如果不指定则不进行压缩
convert astronaut.png -compress lzw -define tiff:tile-geometry=256x256 astronaut.2.tif
tiffinfo astronaut.2.tif
=== TIFF directory 0 ===
TIFF Directory at offset 0x82f46 (536390)
  Image Width: 512 Image Length: 512
  Tile Width: 256 Tile Length: 256
  Bits/Sample: 8
  Compression Scheme: LZW
  Photometric Interpretation: RGB color
  FillOrder: msb-to-lsb
  Orientation: row 0 top, col 0 lhs
  Samples/Pixel: 3
  Planar Configuration: single image plane
  Page Number: 0-1
  White Point: 0.3127-0.329
  PrimaryChromaticities: 0.640000,0.330000,0.300000,0.600000,0.150000,0.060000
  Predictor: horizontal differencing 2 (0x2)
openslide-show-properties astronaut.2.tif
openslide.level-count: '1'
openslide.level[0].downsample: '1'
openslide.level[0].height: '512'
openslide.level[0].tile-height: '256'
openslide.level[0].tile-width: '256'
openslide.level[0].width: '512'
openslide.quickhash-1: 'e6e70ecf3d6f665bc6049487081fee658b58f61de4d02ea37dbbc6f2c8ae6cb3'
openslide.vendor: 'generic-tiff'
tiff.ResolutionUnit: 'inch'
```

可以看到，openslide 可以正确识别了这个 tiff 文件，并且 `tiffinfo` 和 `openslide-show-properties` 的输出都显示出了 tile width 和 tile height 的信息．

imagemagick 中还可以指定输出为 pyramid tiff，命令如下：

```bash
convert astronaut.png -compress lzw -define tiff:tile-geometry=256x256 ptif:astronaut.3.tif
```

但是这个命令生成的金字塔只有一层．我们可以使用 `-define ptif:pyramid=min-basexlevels` 来指定金字塔图像最小那层的尺寸以及层数，命令如下：

```bash
convert astronaut.png -compress lzw -define tiff:tile-geometry=256x256 -define ptif:pyramid=64x4 ptif:astronaut.4.tif
identify astronaut.4.tif
astronaut.4.tif[0] TIFF 512x512 512x512+0+0 8-bit sRGB 739324B 0.000u 0:00.000
astronaut.4.tif[1] TIFF 256x256 256x256+0+0 8-bit sRGB 0.000u 0:00.000
astronaut.4.tif[2] TIFF 128x128 128x128+0+0 8-bit sRGB 0.000u 0:00.000
astronaut.4.tif[3] TIFF 64x64 64x64+0+0 8-bit sRGB 0.000u 0:00.000
openslide-show-properties astronaut.4.tif
openslide.level-count: '1'
openslide.level[0].downsample: '1'
openslide.level[0].height: '512'
openslide.level[0].tile-height: '256'
openslide.level[0].tile-width: '256'
openslide.level[0].width: '512'
openslide.quickhash-1: 'e6e70ecf3d6f665bc6049487081fee658b58f61de4d02ea37dbbc6f2c8ae6cb3'
openslide.vendor: 'generic-tiff'
tiff.ResolutionUnit: 'inch'
```

可以看到 `identify` 命令已经确认这个是一个金字塔图像，但是 openslide 只能识别到金字塔的一层．根据 [issue 4464](https://github.com/ImageMagick/ImageMagick/issues/4464)，这可能是 imagemagick 的一个 bug．

要创建 tiled pyramid tiff，我们还可以使用 [libvips](https://github.com/libvips/libvips)．他提供有命令行工具以及 Python 包．简单起见，我们这里直接使用 vips 的命令行工具．转换命令如下：

```bash
vips tiffsave astronaut.png astronaut.5.tif --compression=lzw --tile --tile-width=256 --tile-height=256 --pyramid
# 查看信息
openslide-show-properties astronaut.5.tif
openslide.level-count: '2'
openslide.level[0].downsample: '1'
openslide.level[0].height: '512'
openslide.level[0].tile-height: '256'
openslide.level[0].tile-width: '256'
openslide.level[0].width: '512'
openslide.level[1].downsample: '2'
openslide.level[1].height: '256'
openslide.level[1].tile-height: '256'
openslide.level[1].tile-width: '256'
openslide.level[1].width: '256'
openslide.mpp-x: '352.86108382487424'
openslide.mpp-y: '352.86108382487424'
openslide.quickhash-1: '64371afc8c9ea13c7d19411a390645d062b79c7c563df2ca4cab577346a09d79'
openslide.vendor: 'generic-tiff'
tiff.ResolutionUnit: 'inch'
tiff.XResolution: '71.983001708984375'
tiff.YResolution: '71.983001708984375'

tiffinfo astronaut.5.tif
=== TIFF directory 0 ===
TIFF Directory at offset 0x82f46 (536390)
  Image Width: 512 Image Length: 512
  Tile Width: 256 Tile Length: 256
  Resolution: 71.983, 71.983 pixels/inch
  Bits/Sample: 8
  Sample Format: unsigned integer
  Compression Scheme: LZW
  Photometric Interpretation: RGB color
  Orientation: row 0 top, col 0 lhs
  Samples/Pixel: 3
  Planar Configuration: single image plane
  Predictor: horizontal differencing 2 (0x2)

=== TIFF directory 1 ===
TIFF Directory at offset 0xa6060 (680032)
  Subfile Type: reduced-resolution image (1 = 0x1)
  Image Width: 256 Image Length: 256
  Tile Width: 256 Tile Length: 256
  Resolution: 71.983, 71.983 pixels/inch
  Bits/Sample: 8
  Compression Scheme: LZW
  Photometric Interpretation: RGB color
  Orientation: row 0 top, col 0 lhs
  Samples/Pixel: 3
  Planar Configuration: single image plane

identify astronaut.5.tif
astronaut.5.tif[0] TIFF 512x512 512x512+0+0 8-bit sRGB 680252B 0.000u 0:00.000
astronaut.5.tif[1] TIFF 256x256 256x256+0+0 8-bit sRGB 0.000u 0:00.000
```

可以看到，这里无论使用 `identify`、`tiffinfo` 还是 `openslide-show-properties` 都可以正常查看到生成的 tiled pyramid tiff 文件是有 2 层金字塔的．

## 如何读写大尺寸的 tiff 图像

对于病理图像，我们还需要考虑一个问题，如果将整个图像全部载入内存会导致 OOM，那我们应该如何处理？常用的 openslide 库仅支持读取病理图像．我们可以根据需要在指定的金字塔层级中读取指定的一块区域，进行图像分析处理，最后把结果保存到磁盘中．但是，openslide 并不支持写入 tiff 文件，特别是不支持写入 tiled pyramid tiff 文件．因此，我们还需要一个支持读写 tiled pyramid tiff，至少是支持读写 tiled tiff 的图像处理库．因为很多已有的程序一般都用 openslide 来进行分析，我们还是希望我们处理之后的 tiff 文件能够继续给 openslide 处理，从而保证我们的处理流程不需要太多的修改．

### rasterio

[rasterio](https://github.com/rasterio/rasterio) 就是其中一个比较不错的选择．他支持按照 window 来读取和写入 tiff 文件，从而保证不会遇到 OOM 的问题．例如，以下代码对输入图像进行裁剪，保留感兴趣的区域，然后输出为 tiled pyramid tiff 文件：

```python
import argparse

import rasterio
from rasterio.profiles import DefaultGTiffProfile
from rasterio.windows import Window
from tqdm import tqdm


def get_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", type=str, required=True, help="input image file")
    parser.add_argument("--output", type=str, required=True, help="output image file")

    return parser.parse_args()


if __name__ == "__main__":
    args = get_args()
    filename = args.input
    with rasterio.open(filename) as src:
        # 计算裁剪的部分
        h, w = src.shape
        top = 79000
        bottom = top + 10240
        left = 24500
        right = left + 10240
        block_size = 1024

        height = bottom - top
        width = right - left

        # RGB 图像需要设置 count = 3
        # width 和 height 不在默认的 profile 中，需要根据实际设置
        profile = DefaultGTiffProfile(count=3, width=width, height=height)
        print(f"writing tiff with profile: {profile}")

        # 保存裁剪后的图像
        with rasterio.open(args.output, "w", **profile) as dst:
            # 逐块写入
            for i in tqdm(range(top, bottom, block_size)):
                for j in range(left, right, block_size):
                    block_height = min(i + block_size, bottom) - i
                    block_width = min(j + block_size, right) - j
                    src_window = Window(j, i, block_width, block_height)
                    block_data = src.read(window=src_window)
                    dst_window = Window(j - left, i - top, block_width, block_height)
                    dst.write(block_data, window=dst_window)
            # 创建图像金字塔
            dst.build_overviews([2, 4, 8, 16])
```

这里的 tiled 是由 `DefaultGTiffProfile` 中的 `tiled = True` 设置的，而图像金字塔则是在图像数据写入完毕之后，调用的 `build_overviews` 方法来追加的．如果不需要图像金字塔，只要不调用此方法即可．另外，由于 rasterio 在这里是逐 window 处理的，window 的尺寸可以根据计算资源而调整，从而避免了 OOM 的问题．

## vips

[libvips](https://github.com/libvips/libvips) 是另外一个不错的选择，libvips 是一个快速的图像处理库，且它可以以很低的内存需求运行．它同时也提供了 Python 接口，即 [pyvips](https://github.com/libvips/pyvips)．libvips 将如何处理大尺寸图像的细节给隐藏起来，用户只需要使用 libvips 的各种操作，而无需关心底层是如何处理的．这也是它的一个比较大的优点．我们不需想像 rasterio 那样逐个 window 的处理图像，这部分细节已经由 libvips 为我们处理．对于上面同样的裁剪图像的例子，我们可以直接使用以下 python 代码：

```python
import argparse

import pyvips


def get_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", type=str, required=True, help="input image file")
    parser.add_argument("--output", type=str, required=True, help="output image file")

    return parser.parse_args()


if __name__ == "__main__":
    args = get_args()
    left = 24500
    top = 79000
    width = 10240
    height = 10240

    image = pyvips.Image.new_from_file(args.input)
    image = image.crop(left, top, width, height)
    # 注意，这里我们设置 bigtiff=True，避免图像太大超过 4GB 的时候出错
    # 如果是使用 vips tiffsave 命令，也可以加上 --bigtiff 选项
    image.tiffsave(
        args.output,
        tile=True,
        tile_width=256,
        tile_height=256,
        pyramid=True,
        bigtiff=True,
        compression=pyvips.enums.ForeignTiffCompression.LZW,
    )
```

可以看到，我们这里只需要使用 pyvips 读取图像，然后调用 `crop` 方法裁剪，最后保存图像．我们并不需要关心裁剪的区域太大内存不够怎么办，这些都由 libvips 在底层为我们处理好．如前文所言，libvips 也有提供命令行工具，我们也可以使用其命令行工具，先进行裁剪，然后再转换为 tiled pyramid tiff．但是由于命令行不支持链式操作，我们只能将原图裁剪保存为中间文件，然后再转换格式，多出来的 IO 操作会增加一定的耗时．

## 总结

综上所述，如果使用 Python 代码处理病理图像的话，如果需要进行读写，且要求写入 openslide 能够读取的 tiled tiff 文件，则推荐使用 pyvips 来出来．如果只是简单的快速处理，也可以直接使用 libvips 的命令行工具，而无需编写 Python 代码．这样处理之后可以保存为 openslide 支持的格式，方便后续的数据处理．

如果还需要给特定的图像查看器来查看，一般还要求这个 tiled tiff 文件包含图像金字塔，也就是 tiled pyramid tiff，这些也都能由 libvips 或者 pyvips 解决．对于文件尺寸大于 4G 的，还需要使用 bigtiff．简单起见，我们可以直接要求 libvips 或者 pyvips 输出 tiled pyramid bigtiff．

此外，tiff 文件的压缩方式，一般可以考虑使用无损压缩 lzw．如果能够接受有损压缩，则 JPEG 编码压缩会有更好的压缩率，默认的压缩质量设置为 75，也可以根据自己的需要将压缩质量设置为 90 或者 95，能保留更多的图像细节．
