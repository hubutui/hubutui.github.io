---
title: "VS code 中调试 R 语言脚本"
date: 2021-04-02T00:00:00+08:00
---

我们之前讲过了如何使用 conda 来管理 R 语言环境．本文，我们来讲讲如何使用 VS code 来调试 R 语言脚本．

实际上我们只是需要：

1. 安装 VS code 插件
   1.  [vscode-r](https://marketplace.visualstudio.com/items?itemName=Ikuyadeu.r)
   2. [vscode-r-lsp](https://marketplace.visualstudio.com/items?itemName=REditorSupport.r-lsp)
   3. [R Debugger](https://marketplace.visualstudio.com/items?itemName=RDebugger.r-debugger)
2. 安装 R 语言包
   1. [languageserver](https://github.com/REditorSupport/languageserver): `conda install -c conda-forge r-languageserver `
   2. [vscDebugger](https://github.com/ManuelHentschel/vscDebugger): `conda install -c conda-forge r-devtools`，然后启动 R 命令行，`devtools::install_github("ManuelHentschel/vscDebugger/")`

安装完毕即可在 R 脚本上打断点，运行，调试了．