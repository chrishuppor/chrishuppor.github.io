---
layout: post
title: "Reversing.kr_CSHARP"
date: 2019-6-3 16:46:53
categories: hello
tags: hello
---


Reversing.kr的CSHARP，是一个动态解密并调用函数的程序。


# Reversing.kr_CSHARP

## 解题过程

1. 拖入PEiD得知这是一个C#程序。
2. 拖入dnSpy，发现这个程序还是很简单的，只有一个Form1的类。但是这个类中有一个成员函数MetMett无法解析，肯定有猫腻。
3. 从按键响应函数入手，发现输入的数据被传到MetMetMet函数中处理，并且该函数会给出最终的匹配结果。所以关键逻辑就在这个函数里面了。
4. 
