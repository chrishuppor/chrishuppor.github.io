---
layout: post
title: "Reversing.kr_FlashEncrpyt"
date: 2019-5-30 19:21:16
categories: WriteUp
tags: Reversing_kr
---


Reversing.kr的FlashEncrpyt，一个地地道道的Misc签到题。



# Reversing.kr_FlashEncrpyt

## 解题过程

1. 将文件拖入flash分析工具FFDec.

   如下，看到了很多东西，尽管不懂flash，但凭经验也知道关键在于脚本。

   ![图1 加载swf](https://chrishuppor.github.io/image/Snipaste_2019-05-30_16-35-45.PNG)

2. 查看脚本，发现其中的代码很混乱，看来是加了混淆。FFDec也提示使用设置中的自动去混淆功能。

   ![图2 自动去混淆](https://chrishuppor.github.io/image/Snipaste_2019-05-30_16-40-57.PNG)

3. 去混淆后，脚本代码就变得十分清晰了。用户输入数据到文本框中，点击按钮触发脚本。如果用户输入的数据通过匹配，则调用函数转到对应的帧。如下，如果输入是1456，点击按钮后则转到frame3.

   ![图3 脚本代码示例](https://chrishuppor.github.io/image/Snipaste_2019-05-30_16-42-36.PNG)

4. 从脚本中可以得知每一帧对应的数字，从第一帧开始逐个输入，最终会转到frame7，并显示出Flag.

## 启示

* FFDec确实是个好使的flash分析软件
* 不管什么类型的文件，运行在什么平台，使用什么语言写的，逆向的本质就是找出功能与代码的对应关系。