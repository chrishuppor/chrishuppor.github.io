---
layout: post
title: "windows下pintools的编译与使用"
pubtime: 2019-09-20
updatetime: 2020-09-20
categories: Reverse
tags: tools pin
---

记录pin的安装、pintools的编译与使用。

# pin安装

在pin的官网[http://pintool.org/ ](http://pintool.org/)下载最新版本的pin工具包，注意要和自己机器的系统匹配。

下载下来的就是pin的exe，不需要编译，可以直接使用；需要自己编译的是pintools。

# pintools的编译与使用

pintools本质就是一个动态链接库。pintools的编译就是用c/c++的编译工具将tools源码编译链接成dll(windows下)。

## 示例

以pin自带的示例inscount0为例，进行编译与使用。(路径："%Pin安装目录%\source\tools\ManualExamples")

（据说要先安装cygwin，但我的机器里本来就装了cygwin，而且编译pintools时也没有用到，所以也不知道是不是真的要装）

**编译**

1. 打开vs适合操作系统版本的**命令行**，进入"%Pin安装目录%\source\tools\ManualExamples"
2. 执行`make obj-intel64/inscount0.dll`，则会在"%Pin安装目录%\source\tools\ManualExamples\obj-intel64"下生成inscount0.dll，这个就是我们要用的pintools.

**使用**

执行```<pin.exe的路径> -t <pintools的dll路径> -- <被分析的程序的路径>```
需要注意的是，dll与被分析的程序的位数要一致。

---

题外话：linux下pintools的注意事项：

linux下pin安装要对应好版本（gcc版本和系统位数），否则在编译时就是一堆堆的err。以下是我的个人经验：

* 在ubuntu 18.04 64位机器上安装pin-3.7，可以编译目标平台为intel64的程序，不可以编译ia32的程序。
* 在ubuntu 16.04 32位机器上安装pin-2.7会因为gcc版本不匹配报错；安装pin-3.7，十分舒适。