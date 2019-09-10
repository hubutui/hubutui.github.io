---
title: "Qt5 Notepad"
date: 2017-08-21T00:00:00+08:00
---

# Qt 5 自带的 Notepad 示例

## 创建工程

1. File > New File or Project > Applications > Qt Widgets Application > Choose
1. Name: Notepad, Create in `/home/darcy/Projects/qt`
1. Class Information 中，Base class 选择 `QMainWindow`，Class name 改为 `Notepad`．可以注意到相应的头文件等都自动改名了．

继续点下一步完成工程的创建．可以看到 Qt Creator 为我们自动创建了一些必备的文件：

* `notepad.pro` - 工程文件
* `main.cpp` - 主函数在该文件中
* `notepad.cpp` - `Notepad` widget 类的源文件
* `notepad.h` - `Notepad` widget 类的头文件
* `notepad.ui` - `Notepad` widget 类的 UI 文件

## 详细的介绍这些文件

### main.cpp

```cpp
#include "notepad.h"
#include <QApplication>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    Notepad w;
    w.show();

    return a.exec();
}
```

这个文件的前两行是头文件的包含，每一个 Qt 类都有一个同名的头文件，这里用的是 `QApplication`，自然就应该包含 `QApplication` 的头文件了．

然后是 `main` 函数中，首先构建了一个 `QApplication` 的对象 `a`, 并给出了初始化列表，也就是 `main` 函数的参数．

再接着的一行构建了一个 `Notepad` 类的对象 `w`．在 Qt 中，包含各种可视化元素的用户界面称为 widget，例如各种标签，按钮等等．

widget 默认是不显示的，所以接着的一行调用 `Notepad` 的 `show()` 函数，将其显示出来．

最后一行就是返回语句，它使得程序进入事件循环．

### UI 设计

双击打开 `notepad.ui` 文件，即可直接打开 Qt Designer．在 Qt Designer 中，我们可以方便的使用拖拽的方式添加各种部件．

这里我们添加一个 `Text Edit` 和一个 `Push Button`．添加好之后双击 `Push Button`，输入“退出”．选中 `Push Button`，将其 `objectName` 改为 "quitButton"．选中这两个控件，选择 Lay out Vertically，或者直接按快捷键 `CTRL + L`．这个布局的修改是使得程序进行放大缩小的时候，控件的大小和位置也能够随之改变．

此时你可以看看 `notepad.ui` 文件的内容，这是个 xml 文件，用来描述界面的．文件内容会越来越多．考虑到篇幅所限，这里不列出了．

### notepad.h 头文件

Qt Creator 创建的 `notepad.h` 文件中已经有了一些必要的内容，如一个构造函数，一个析构函数，相应的 UI 对象等等．详细的可以自己看看该文件．这里不列出．

### notepad.cpp 源文件

这个文件的内容暂时比较简单．没什么好说的．

### notepad.pro 工程文件

工程文件最主要的目的是用来给 `qmake` 生成 `Makefile` 的．这里不多介绍，有兴趣可以参考 `qmake` 的文档．

## 添加用户交互

就目前来说，我们的工程是可以构建了．但是运行起来没有任何用处，因为还没有添加用户交互．比如说，你怎么点那个界面上的退出按钮都不会有任何反应的．

Qt 中使用独有的信号与槽 (signals & slots) 机制来进行事件处理．当一个事件发生，它会触发一个信号，这个信号被一个槽函数接收到，由这个槽函数去响应这个信号．

### Push Button 的交互

这里用具体的例子来说明，显然我们希望用户点击退出按钮的时候，程序就退出．在 Qt Designer 中，选中退出按钮，右键选择 Go to slot > `clicked()`, `notepad.h` 头文件中就会添加好一个私有的 slot: `on_quitButton_clicked`，它的实现 `Notepad::on_quitButton_clicked()` 会添加到 `notepad.cpp` 中，目前该函数体的内容还是空的，需要我们补充．因为我们要在用户点击退出按钮的时候退出程序，所以这个函数内容只要写一句退出语句即可：

```cpp
void Notepad::on_quitButton_clicked()
{
    QApplication::quit();
}
```

重新构建下项目，然后尝试运行，这回点击退出按钮即可退出程序了．

### 更多的用户交互

一般地，一个文本编辑器应该有打开文件，保存文件等等内容吧．我们需要添加菜单栏．在 Qt Designer 中，Notepad 的 UI 界面上有个"Type here"的占位符，双击即可在该处添加文件并创建一个菜单．

## 参考

* [Getting Started Programming with Qt Widgets](http://doc.qt.io/qt-5/gettingstartedqt.html)
