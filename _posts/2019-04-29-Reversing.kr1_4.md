---
layout: post
title: "Reversing.kr"
date: 2019-04-29 8:8:8
categories: WriteUp
tags: Reversing.kr 
---

微简RE challenge网站，my_4ear_3hr1s 就系我啦，1-4题解题过程及题后思考记录如下。

# Reversing.kr_1-4

## 1. Easy_CrackMe.exe

RE除了技术还有很大一方面是**社会工程学：分析编写者和编写背景。**编写者的习惯、擅长的技术以及编写的目的都会告诉我们应该去哪里找我们想要的东西以及如何找。

一般题目都是越来越难，所以这个题目作为网站的第一题，一定十分简单，也就是说没有壳、混淆、反调等技术，也不会有各种隐藏。作为验证码程序，也不会使用复杂的验证逻辑和算法，一定是简单到开门见山的。

### 破解过程

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

### 小结

* 这个验证码匹配使用的最简单的方法——字符串比较。只不过没有将完整的字符串直接写在程序中，而是将字符串拆开进行分段比较。
* 分析时会遇到很多项v3 v4 v5这种看上去不清不楚的变量，有时只需注意他们在栈中的位置关系就可以知道他们分别代表什么。
* 由API调用入手，找到关键位置。

## 2. Easy Keygen.exe

### 破解过程

1. 了解程序功能

   输入name和serial，如果不匹配就输出wrong

   ![程序功能](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_16-32-9.PNG)

2. 猜测关键点

   比较之后输出wrong，所以找到输出wrong的代码，其前面就是比较的逻辑。

   wrong应该是使用printf等函数输出的，可以先定位wrong字符串，然后查看其引用就可以找到输出的代码。

3. 查壳

   没壳（其实想也知道没壳）

   ![查壳](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_16-35-54.PNG)

4. 拖进IDA

   1. 查看string，找到wrong

      ![string](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_16-39-27.PNG)

   2. 查看wrong的引用

      ![Snipaste_2019-04-29_16-41-09.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_16-41-09.PNG)

   3. wrong唯一的引用在main函数中，main函数流程如下：

      1. 接收name输入

      2. 处理name字符串

         依次将字符与指定字符异或，并以“%02X”的形式输出到v13中

         ![Snipaste_2019-04-29_16-53-37.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_16-53-37.PNG)

      3. 接收serial

      4. 比较serial和处理后的name

         ![Snipaste_2019-04-29_16-54-56.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_16-54-56.PNG)

5. readme要求获得指定serial对应的name，所以需要自行编写name处理逻辑的逆算法

   ```python
   aSerial = "5B134977135E7D13"
   Clen = int(len(aSerial)/ 2)
   j = 0
   aName = ''
   XorList = [16, 32, 48]
   for i in range(0, Clen):
       tmp = aSerial[i*2: i*2 + 2]
       integer = int(tmp, base = 16) #注意：处理name时是按十六进制输出到字符串的，所以逆算法也要先把字符串变为十六进制整数
   
       if j >= 3:
           j = 0
           
       aName = aName + chr(integer ^ XorList[j])
       j = j + 1
   
   print (aName)
   ```

### 小结

* 这个验证码匹配使用的是第二阶的验证方法——比较计算结果。不进行字符串比较，而是将输入的字符串进行一系列计算，然后将计算结果与正确的值进行比较。这种方法的破解难度与获得正向算法的逆算法的难度相关。
* 由提示字符串入手，找到关键位置。

## Easy Unpack.exe

这是个简单的带壳程序，会被杀软查杀，因此分析时需要关闭杀软，关闭windows defender

### 破解过程

1. 查看ReadMe

   要求找到OEP（说明这是个加壳文件）

2. 查壳

   使用PEiD查看程序，没有发现喜闻乐见的壳，看来只能自己找。

   ![Snipaste_2019-04-29_21-52-09.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-04-29_21-52-09.PNG)

3. 拖进OD，使用OD自解压直接获得OEP

### 小结

* 使用OD自解压功能
* 也可以自己看，这个壳十分简单，脱壳后有一个明显的JMP

## Music Player

### 破解过程

1. 运行程序

   这是一个MP3播放程序，将MP3文件播放一分钟后会弹出一个提示框。Readme要求绕过“一分钟”的限制。

   猜测关键位置应该是一个cmp和je系列指令的组合——先比较时间，然后进行跳转：如果不到一分钟就继续播放，过了一分钟就进行检测，通过检测就给出Flag，不通过就弹出提示框。

   ![程序](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_19-24-56.PNG)

2. 拖进PEiD

   这是个VB程序，没有加壳。

   ![Snipaste_2019-05-04_19-08-05.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_19-08-05.PNG)

3. 拖进VB Decompile

   如图，可以看到两个带有“TIMER”字样的函数。既然破解目的是突破时间限制，那么这很可能是关键代码所在位置。查看其它函数，发现确实没有感兴趣的地方。

   在TMR_POS_Timer中，可以看到一个与60000的比较。注意到60000ms = 60s，所以这里应该就是时间控制的跳转——如果时间大于60s，则停止播放，并弹出提示框。但这不是Flag的提示框，所以我们要控制程序直接跳到4045FE。

   ![Snipaste_2019-05-04_19-37-58.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_19-37-58.PNG)

   到此，其他地方没有看出问题，直接拖进OD吧。

4. 拖进OD

   1. 修改0x40456B的跳转，使其直接跳转到0x4045FE。删除0x40456b断点，继续运行程序，看看会发生什么。（需要忽略所有异常才能正常运行。）如图，成功突破60s的限制，但是没有弹出Flag，而是弹出来一个运行异常提示框。

      ![Snipaste_2019-05-04_20-12-33.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_20-12-33.PNG)

      弹出这个提示框后，点击确定，程序会退出。在程序中搜索字符串，也无法找到相关字符。经过Google得知这个提示框是windows系统弹出的，用于提示用户发生了异常。

      保存修改的程序。

   2. 如果想弹出Flag，就不能弹出这个异常，就需要知道这个异常是谁抛出的，然后绕过它。

      需要在RaiseException处下断点，运行修改程序。程序中断后，查看堆栈中函数的返回地址，依次查询直到该地址落在主模块领域。

      结果发现RaiseException的触发在0x4046B9的vbaHresultCheckObj函数调用中。

      经查询，vbaHresultCheckObj函数是一个对象检查函数，如果该对象没有通过检查则触发异常。如图，在vbaHresultCheckObj上方有一个条件跳转，如果满足条件则会跳过vbaHresultCheckObj调用，现在直接将其修改为jmp。

      ![Snipaste_2019-05-04_23-04-10.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_23-04-10.PNG)

      保存修改结果，运行修改后的程序。

5. Flag

   如图，Flag不是用MsgBox弹出，而是使用修改对话框标题的方式给出的。这就防止破解者通过查找MsgBox调用地址直接找到关键位置。



![Flag](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_18-59-44.PNG)

### 小结

在破解程序时，遇到了如下困难：

* P-CODE模式的VB程序的反编译

  * 之前没有接触过VB程序——既没有编写过也没有破解过，无所适从。

    最初用IDA根本看不到有意义的伪代码，IDA也无法识别函数。经过搜索才知道，VB程序有两种编译模式，伪代码编译的VB程序使用一种特殊的伪代码，运行时该程序会运行在VB虚拟机中。IDA难以自动反编译这种模式的程序。

  * 解决办法

    * 如果IDA不能识别一个函数，可以手动告诉IDA这里是一个函数，在这个地址上按下 P 键，再使用 F5 就可以看到伪代码了。但是这个需要知道哪些汇编代码是一个函数里的。
    * 使用VB Decompiler工具

* Flag的定位

  * 此前接触的Flag往往以printf或MsgBox的方式给出，因此尝试使用MsgBox定位到Flag的方法不再适用

在寻找关键位置的过程中进行过如下尝试：

* 在查找MsgBox调用位置时，由于调用列表很长，直接从调用列表中查找有些困难，因此采用了查看返回地址的方法：

  1. 如图查找MsgBox地址，在函数入口地址下断点。

  ![Snipaste_2019-05-04_21-19-46.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_21-19-46.PNG)

  2. 继续运行程序，中断在MsgBox入口，此时转到栈中返回地址就可以找到一个MsgBox的调用。

     ![Snipaste_2019-05-04_21-25-29.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-04_21-25-29.PNG)

  3. 指向该调用，右键“查找引用->调用目标”就可以查看该函数的所有调用了。