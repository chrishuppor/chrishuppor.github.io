---
layout: post
title: "Reversing.kr_AdventureCustomTips"
date: 2019-7-26 16:8:7
categories: Reverse
tags: WriteUp
---

最后两个题逻辑不是很难，难点在于调试——一个需要AVR环境，一个要用到plmdebug。只要找对了调试的方法，就相当于破解了，剩下的就很简单了。


# Adventure

## 调试

与MetroApp一样，是一个appx程序。微软提供了appx的调试工具plmdebug，这个工具能够开启appx程序的调试状态并指定调试器（我使用的x32dbg，因为OD总是报错），可以在winSDK中获得。

## 程序逻辑

本程序是一个游戏，每打一个怪可以获得一分，当获得一定分数时程序会显示出flag。

1. 可以爆破掉打怪部分的代码，使程序自动打怪。大概一个小时候就可以获得第一个flag，大概595天后就可以获得第二个flag，所以可以等到一年多之后再来提交flag。

   所以最终还是要破解flag生成逻辑，而第一个flag主要就是给我们用来做验证的。

2. flag初始值为0，每打一个怪就会对flag进行一些操作，而操作数由rand函数获得。记得rand函数生成的随机数是伪随机数吗？去研究一下吧。

其他请参考：[bendawang](http://www.bendawang.site/2019/06/23/reversing_kr%20writeup[26]/)

# Custom

## 调试

这是一个AVR架构的单片机程序，需要AVR Studio和hapsim进行动态调试。可以用IDA看汇编代码，需要选择ATmega128平台。

## 程序逻辑

程序是一个类似于linux的系统，把flag加密存储在其中的readme中，通过用户输入来解密，然后就可以使用cat查看了。

1. 用户名可以直接找到，但是pw无法在内存中直接找到。因为程序对用户输入的pw进行了一些运算，需要爆破一下。

2. 直接查看readme时会提示没有这个文件，因为程序中只有"readme "这个文件，但系统不接受用户输入中的空格，所以直接爆破掉就可以。

其他请参考：[bendawang](http://www.bendawang.site/2019/06/23/reversing_kr%20writeup[27]/)

# 后记

之前的题可以轻松的在网上找到各路大佬写的wp和flag，所以我也公开了自己的wp。在做CRC2时，我也是参考了大佬的wp才学会解题方法的。

这两个题有所不同。Custom的wp网上可以找到，flag也可以找到，但不多；adventure的wp没有，但一些大佬给出了hint。我这里也是只给hint了，因为复盘的乐趣毕竟没有自己动手的乐趣足。除此之外，能不能挤到rank的首页，关键就是这两个题了，如果给出了可以轻松获得flag的方法，我将无法心安理得的面对与之前辛苦做题打排名的人。