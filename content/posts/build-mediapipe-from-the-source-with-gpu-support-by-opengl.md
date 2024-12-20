---
title: "Linux 下编译安装 OpenGL GPU 支持的 mediapipe 指南"
date: 2024-02-22T00:00:00+08:00
toc: true
images:
tags:
  - mediapipe
  - deep learning
  - archlinux
  - python
---

## 简介

[mediapipe](https://github.com/google/mediapipe) 是由 Google 开发的适用于直播和流媒体的跨平台、可定制的机器学习解决方案．他提供了人脸识别、人脸关键点检测、手势识别等各种算法，支持 iOS, Android, PC 等多种平台，支持 C++, JAVA, Python 等多种语言．

这里我们主要关注的是 Linux 桌面平台和 Python 语言部分．

## 安装

CPU 版本可以直接从 PyPI 安装，使用命令 `pip install mediapipe` 即可．

## GPU 支持与安装

根据[官方文档](https://developers.google.com/mediapipe/framework/getting_started/gpu_support)的说明，mediapipe 的 GPU 支持可以使用 OpenGL 或者 CUDA 来完成．本文仅提供使用 OpenGL 的说明，通过 CUDA 支持的，可以参考：

1. [Mediapipe 0.10.1 with CUDA GPU Support python libs](https://github.com/pydehon/mediapipe)，这个需要在 Docker 中编译，且仅限 mediapipe 0.10.1 版本，cuda 和其他包的版本也受限于 Docker 基础镜像．
2. [MediaPipe CPU + GPU 的安装和使用](https://blog.csdn.net/qq_56548850/article/details/123981579)，这个主要是针对 Jetson Nano，很有参考价值．

另外说明一下，这里说的 GPU 支持，并不仅限于 Nvidia 显卡．当然，使用 CUDA 则仅限 Nvidia 显卡．而使用 OpenGL，则可以支持使用 Nvidia, AMD 和 Intel 显卡，只需安装好对应的显卡驱动包，且显卡支持 OpenGL 3.1 及以上版本即可．

### 编译安装

1. ArchLinux 用户可以直接使用 [python-mediapipe-pkgbuild
   ](https://github.com/hubutui/python-mediapipe-pkgbuild) 提供的 PKGBUILD 文件来打包，目前支持版本为 0.10.9．github 上的此仓库缺乏维护，建议使用 AUR 上的版本．你也可以从 AUR 上安装 [python-mediapipe-git](https://aur.archlinux.org/packages/python-mediapipe-git) 或者 [python-mediapipe](https://aur.archlinux.org/packages/python-mediapipe)．
2. 其他发行版的用户可能需要根据实际需要，对 mediapipe 的源码做一些简单的修改，详细的 patch 都已经在 [python-mediapipe-pkgbuild
   ](https://github.com/hubutui/python-mediapipe-pkgbuild) 中提供．这里列出需要注意和修改的点：

   - 修改 [WORKSPACE](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/WORKSPACE#L89-L91)，将 `rules_apple` 更新到 3.2.1 版本，注意需要自己下载更新后的源码，计算 sha256sum 并更新．20241220 更新，此步骤可以不用了．
   - 修改 [WORKSPACE](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/WORKSPACE#L63-L68)，注释掉应用 `com_google_protobuf_fixes.diff` 补丁的部分．20241220 更新，此步骤可以不用了．
   - 如果你的编译器是 gcc >= 13，需要把第三方包 `com_google_audio_tools`打个补丁，添加 `cstdint` 文件头．这里可以直接根据 [WORKSPACE](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/WORKSPACE#L229) 去下载 com_google_audio_tools 的源码，应用补丁 [com_google_audio_tools_fixes.diff](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/third_party/com_google_audio_tools_fixes.diff)，编辑 [audio/dsp/porting.h](https://github.com/google/multichannel-audio-tools/blob/80892ee5252829701db4e57c9ecc3a825fa1e87c/audio/dsp/porting.h#L23)，加上一行 `#include cstdint`，然后重新生成补丁，覆盖掉 [com_google_audio_tools_fixes.diff](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/third_party/com_google_audio_tools_fixes.diff) 即可．20241220 更新，此步骤可以不用了，官方已经修复．
   - 修改 [opencv_linux.BUILD](https://github.com/google/mediapipe/blob/master/third_party/opencv_linux.BUILD)，根据你已经安装好的 opencv 的版本和路径修改要使用的头文件路径．
   - 修改 [setup.py](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/setup.py#L230)，给 `protoc` 命令加上一个选项 `--experimental_allow_proto3_optional`．20241220 更新，此步骤可以不用了．
   - 设置 `link_opencv` 为 `True`，这个可以直接 [setup.py](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/setup.py)，简单的查找替换即可．
   - 设置版本号，直接修改 [setup.py](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/setup.py) 中的 `__version__` 即可．
   - 修改 [.bazelversion](https://github.com/google/mediapipe/blob/4237b765ce95af0813de4094ed1e21a67bad2a5f/.bazelversion) 中的版本号为你实际使用的 bazel．如果可以，建议直接使用 mediapipe 源码中 `.bazelversion` 指定的版本．20241220 更新，目前官方使用的是 6.5.0 版本，其他版本不做保证．

3. 最后，执行命令 `MEDIAPIPE_DISABLE_GPU=0 python -m build --wheel --no-isolation` 可以编译生成 wheel 文件，该文件保存在 `dist` 目录下，然后使用 pip 命令安装即可．这里若设置 `MEDIAPIPE_DISABLE_GPU=1` 则会编译 GPU 支持．这里的 GPU 支持指的是通过 OpenGL 来支持．Debian 和 Ubuntu 用户需要安装 `mesa-common-dev libegl1-mesa-dev libgles2-mesa-dev` 或者 `nvidia-utils` 等相关包．

## 使用

由于历史原因，mediapipe 实际上有两种使用方式．

## 传统的 solutions 方法

这个主要是使用 `mediapipe.solutions` 的类和方法，例如：

```python
import cv2
import mediapipe as mp
mp_face_detection = mp.solutions.face_detection

img_file = "demo.png"
with mp_face_detection.FaceDetection(
    model_selection=1, min_detection_confidence=0.5) as face_detection:
  image = cv2.imread(img_file)
  # Convert the BGR image to RGB and process it with MediaPipe Face Detection.
  results = face_detection.process(cv2.cvtColor(image, cv2.COLOR_BGR2RGB))
```

基本上就是创建一个类，并且调用他的 `process` 方法．此方法使用起来比较简单，无需指定模型文件．但是缺点是不够灵活，比如无法指定使用的推理设备，只能用默认的 CPU 推理．
他对应的文档在[这里](https://github.com/google/mediapipe/tree/d2bc9e5ba2d8273cbee6ea3298df3ee579d43c35/docs/solutions)，也可以查看[这里](https://developers.google.com/mediapipe/solutions/guide#legacy)，点击对应算法的 info 链接即可查看．一般不再推荐使用．此处列出仅为了方便用户能够看懂旧的代码．

## 新的 solutions 方法

官方推荐使用[此方法](https://developers.google.com/mediapipe/solutions/guide#available_solutions)．此方法主要使用 `mediapipe.tasks.python` 下提供的各个算法的类，用户需要自己设置参数，然调用对应的类的 `create_from_options` 方法来创建一个实例，再调用其中的方法，如 `detect` 方法等，来得到推理的结果．此外，模型文件也需要用户自行下载．注意，这里可以设置 `delegate=python.BaseOptions.Delegate.GPU`，用于指定推理使用的设备为 GPU，默认为 `delegate=python.BaseOptions.Delegate.CPU`．

```python
import mediapipe as mp
from mediapipe.tasks import python
from mediapipe.tasks.python import vision
import mediapipe as mp

model_path = '/absolute/path/to/face_detector.task'

BaseOptions = mp.tasks.BaseOptions
FaceDetector = mp.tasks.vision.FaceDetector
FaceDetectorOptions = mp.tasks.vision.FaceDetectorOptions
VisionRunningMode = mp.tasks.vision.RunningMode

# Create a face detector instance with the image mode:
options = FaceDetectorOptions(
    base_options=BaseOptions(model_asset_path='/path/to/model.task', delegate=python.BaseOptions.Delegate.GPU),
    running_mode=VisionRunningMode.IMAGE)
with FaceDetector.create_from_options(options) as detector:
  img_file = "demo.png"
  mp_image = mp.Image.create_from_file(img_file)
  result = detector.detect(mp_image)
```
