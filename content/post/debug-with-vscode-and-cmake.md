---
title: "Windows 下使用 VS code 和 CMake 进行调试的正确姿势"
date: 2020-05-04T00:00:00+08:00
---

承接[上一篇](/post/how-to-fuckup-msys2-in-windows/)文章，我们介绍了 Windows 下安装和使用 MSYS2 的简单介绍。本文介绍一下如何使用 Visual Studio code 作为开发环境。

VS code 的安装比较简单，这里不再赘述。其他需要在 MSYS2 中安装的包：

```shell
pacman -S mingw-w64-x86_64-toolchain cmake mingw-w64-x86_64-cmake
```

我们还需要设置一下环境变量，将 cmake 添加到 `Path` 中。添加环境变量的方法请自行谷歌，建议添加到用户环境变量，而不是系统环境变量。我们需要添加 `C:\msys64\mingw64\bin`  和  `C:\msys64\usr\bin` 这两个路径进去。如果你的 MSYS2 安装路径不同的话，请自行修改。

下面我们用 vscode + cmake 调试一个来自 vtk 的[例子](https://lorensen.github.io/VTKExamples/site/Cxx/Medical/MedicalDemo1/)，从该页面上下载好该例子的代码（[链接1](https://github.com/lorensen/VTKWikiExamplesTarballs/raw/master/MedicalDemo1.tar)）和数据文件（[链接2](https://raw.githubusercontent.com/lorensen/VTKExamples/master/src/Testing/Data/FullHead.mhd)，[链接3](https://github.com/lorensen/VTKExamples/blob/master/src/Testing/Data/FullHead.raw.gz?raw=true)）。下载后将源码解压，数据文件也解压之后放到源码目录。目录树结构类似这样：

```text
edicalDemo1
├── build
├── CMakeLists.txt
├── FullHead.mhd
├── FullHead.raw
├── MedicalDemo1.cxx
├── MedicalDemo1.java
└── MedicalDemo1.py

```

使用 VS code 打开 `MedicalDemo1` 目录。我们首先安装几个必备的扩展，[cpptools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) 提供 C/C++ 语言支持，[cmake](https://marketplace.visualstudio.com/items?itemName=twxs.cmake) 提供 CMake 语言支持（即 `CMakeFileLists.txt` 里的代码高亮与补全等），[cmake-tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools) 提供 CMake 工具支持（指使用 CMake 构建和调试等）。安装完毕之后，你可能需要重启 VS code，按照提示操作即可。按 `CTRL+SHIFT+P` 打开命令面板，输入 cmake，然后选择 Configure 即可配置项目。所谓配置项目，等价于我们在命令行执行 `cmake -B build -S MedicalDemo1` 命令。首次使用可能需要你配置工具链，也就是告诉 cmake 使用的 gcc 编译器和调试器 gdb 在哪里。这个可以使用命令面板中的 `CMake: Scan for Kits` 命令实现。cmake tools 的工具链配置一般保存在 `C:\Users\USERNAME\AppData\Local\CMakeTools\cmake-tools-kits.json`。你可以打开该文件，然后参考里面已有的工具链进行配置，修改一下名字和相应的编译器路径即可。VS code 下面的状态栏里有一些快捷按钮，包括构建目标类型（对应 `CMAKE_BUILD_TYPE`，我们调试自然应该选择 `Debug`，发布的时候选择 `Release` 即可）、工具链（即编译器等）、生成（对应 `cmake --build build` 命令）、调试。值得一提的是，状态栏里的调试按钮与 Run->Start Debuging（快捷键 `F5`）不是等价的。也许未来会统一吧，可以关注一下[这个](https://github.com/microsoft/vscode-cmake-tools/issues/1202)。

状态栏中的调试按钮可以用于快速调试，但是它功能有很大的限制，比如无法给被调试的目标传参数。如果你的被调试目标需要从命令行接收参数，那么我们需要使用 `F5` 快捷键打开调试。在一个项目中首次使用 `F5` 调试，VS code 会提示你选择一个环境和配置，我们随便选一个。然后会出错，但是它会创建 `.vscode/launch.json` 文件，我们需要用这个文件来进行调试的配置。它的内容大概是这样：

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++.exe - 生成和调试活动文件",
            "type": "cppdbg",
            "request": "launch",
            "program": "${command:cmake.launchTargetPath}",
            "args": ["FullHead.mhd"],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "C:/msys64/mingw64/bin/gdb.exe",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
        }
    ]
}
```

我们需要修改其中的几个参数，`program` 的值改为 `${command:cmake.launchTargetPath}`，这样就可以让 cmake-tools 自动替换为正确的被调试目标的完整路径名；`args` 这个列表里就可以写上被调试目标需要的命令行参数，`miDebuggerPath` 则是调试器 gdb 的完整路径。温馨提示，MSYS2 终端中课可以使用 `cygpath` 进行路径的转换，例如：

```shell
$ cygpath.exe -am /mingw64/bin/gdb.exe
C:/msys64/mingw64/bin/gdb.exe
```

建议使用 `/` 而不是 Windows 里的 `\` 作为路径分隔符号，不然你还需再加一个 `\` 进行转义，写成了难看的这种路径 `C:\\msys64\\mingw64\\bin\\gdb.exe`

Okay，就这么多吧。
