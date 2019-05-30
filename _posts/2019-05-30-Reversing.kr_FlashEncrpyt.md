---
layout: post
title: "Reversing.kr_FlashEncrpyt"
date: 2019-5-30 16:48:31
categories: WriteUp
tags: Reversing_kr
---

Reversing.kr的FlashEncrpyt，一个地地道道的Misc签到题。


# Reversing.kr_FlashEncrpyt

## 解题过程

1. 将文件拖入flash分析工具FFDec.

   如下，看到了很多东西，尽管不懂flash，但凭经验也知道关键在于脚本。

   ![图1 FFDec加载文件](Snipaste_2019-05-30_16-35-45.PNG)

2. 打开一个脚本，发现代码被混淆了，而且FFDec也提示使用设置->自动反混淆功能。

   ![图2 自动反混淆](Snipaste_2019-05-30_16-40-57.PNG)

3. 此时脚本代码已经变得很清晰了。

   首先从输入框获取数据，如果数据通过匹配，则调用 gotoAndPlay函数转到对应的帧。如图，如果输入为1456则转到frame3。

   ![图3 代码](Snipaste_2019-05-30_16-42-36.PNG)

4. 根据脚本代码，从第一帧开始输入，最终会跳转到frame7，此时会显示出Flag

## 启示

* FFDec，一个好使的Flash分析软件。
* 逆向的本质就是能够将功能和代码的对应起来