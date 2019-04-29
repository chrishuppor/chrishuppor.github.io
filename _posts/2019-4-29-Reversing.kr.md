---
layout: post
title: "Reversing.kr"
date: 2019-4-29 16:2:22
categories: WriteUp
tags: REBaby
---

# Reversing.kr

微简RE challenge网站，write up如下。

## 1. Easy_CrackMe.exe

RE除了技术还有很大一方面是**社会工程学：分析编写者和编写背景。**编写者的习惯、擅长的技术以及编写的目的都会告诉我们应该去哪里找我们想要的东西以及如何找。

一般题目都是越来越难，所以这个题目作为网站的第一题，一定十分简单，也就是说没有壳、混淆、反调等技术，也不会有各种隐藏。作为验证码程序，也不会使用复杂的验证逻辑和算法，一定是简单到开门见山的。

1. 了解程序功能

   运行程序，发现这是个验证码程序——输入一个字符串，点击按钮进行验证；如果失败会弹框提示。

   ![程序功能](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_15-28-32.PNG)

2. 猜测关键API

   进行密码验证需要先将字符串读入内存，然后进行比较。

   正确的密码就在比较的逻辑中，因此只需找到比较逻辑所在位置。而比较是在获取字符串后进行的，所以找到获取字符串的位置就找到了比较的位置。

   获取字符串时可能用到GetDlgItemText、GetWindowText等函数。查看其IAT，如下图，发现使用的是GetDlgItemText。因此定位到GetDlgItemText的调用就可以了。

   ![IAT](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_15-32-00.PNG)

3. 拖入IDA

   1. 按G键，输入“GetDlgItemTextA”，跳转到GetDlgItemTextA地址

      ![G](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_15-36-42.PNG)

   2. 按X键，查看GetDlgItemText引用情况，跳转到GetDlgItemText引用位置

      ![X](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_15-38-18.PNG)

   3. 按F5，反编译该段代码

      如下图，红框中表达式如果为真，则弹出Congratulation(通过查看Text可知)；为假则弹出Incorrect Password。可见，这里红框中表达式就是比较的核心逻辑。

      ![核心](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_15-39-38.PNG)

      通过GetDlgItemTextA可知，input_str就是输入的字符串。如下图，根据红线可知，v3在input_str所在地址的下一个位置，&input_str指向第一个字符，&v3指向输入字符串的第二个字符，&v4指向第三个字符，&v5指向第五个字符。比较表达式为真，则要求v3是a, v4是5，*((&v4)+1)是y，&v5指向R3versing，input_str为E——所以输入应为：Ea5yR3versing。

      ![char_ptr](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_15-43-06.PNG)

