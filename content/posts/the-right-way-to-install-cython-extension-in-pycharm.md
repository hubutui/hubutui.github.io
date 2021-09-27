---
title: "PyCharm 中 Cython Extension 的正确编译安装姿势"
date: 2018-12-11T00:00:00+08:00
---

长话短说，正确的命令为：

```shell
python /opt/pycharm-community/helpers/pydev/setup_cython.py build_ext --build-lib ~/.PyCharmCE2018.3/system/cythonExtensions --build-temp ~/.PyCharmCE2018.3/system/cythonExtensions/build
```

其中 `python` 的路径需要根据需要修改，最好应该是你的项目中用的那个 `python`，`setup_cython.py` 的路径也应该根据 PyCharm 的安装路径不同而修改，其他路径同理．

## 略微详细的说

Python 是一门脚本语言，`python` 解释器执行其实很慢的，但是我们可以用 Cython 加速．PyCharm 的 python debugger 支持使用 Cython 加速．*NIX 用户需要注意了，如果你的 PyCharm 安装到系统目录下，直接点击 PyCharm 弹出的提示框而安装 cython extension，默认就直接用 root 权限编译安装到 home 目录去了．想像一下，你的 home 目录下还有个 owner 为 root 的目录？这是什么脑残设计啊．我说的是 `~/.PyCharmCE2018.3/system/cythonExtensions` 目录．或者，如果你按照官网的[提示](https://www.jetbrains.com/help/pycharm/cython-speedups.html)，使用命令安装：

```shell
/usr/bin/python3 /<PYCHARM_INSTALLATION_PATH>/helpers/pydev/setup_cython.py build_ext --inplace
```

那也是有问题的．PyCharm 还是会烦人的提示你 cython extension 可以安装．最终，经过查看 `idea.log` 这个日志文件，才发现了真正的编译安装 cython extension 的命令．就这样子咯．

## emm，独家

目前为止，这个应该是独家的吧．不过反正也没有看到 debug 的时候有多大的提速啦，反正能用，也没有了烦人的提示了．
