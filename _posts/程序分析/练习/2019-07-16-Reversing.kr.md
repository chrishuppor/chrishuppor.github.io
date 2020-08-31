---
layout: post
title: "Reversing.kr"
pubtime: 2019-04-29
updatetime: 2019-07-16
categories: Reverse
tags: WriteUp
---

Reversing.kr@my_4ear_3hr1s，解题过程及题后思考记录如下。

# 1 Easy_CrackMe.exe

RE除了技术还有很大一方面是**社会工程学：分析编写者和编写背景。**编写者的习惯、擅长的技术以及编写的目的都会告诉我们应该去哪里找我们想要的东西以及如何找。

一般题目都是越来越难，所以这个题目作为网站的第一题，一定十分简单，也就是说没有壳、混淆、反调等技术，也不会有各种隐藏。作为验证码程序，也不会使用复杂的验证逻辑和算法，一定是简单到开门见山的。

## 1.1 破解过程

1. 了解程序功能

   运行程序，发现这是个验证码程序——输入一个字符串，点击按钮进行验证；如果失败会弹框提示。

   ![图1 程序界面](https://chrishuppor.github.io/image/Snipaste_2019-04-29_15-28-32.PNG)

2. 猜测关键API

   进行密码验证需要先将字符串读入内存，然后进行比较。

   正确的密码就在比较的逻辑中，因此只需找到比较逻辑所在位置。而比较是在获取字符串后进行的，所以找到获取字符串的位置就找到了比较的位置。

   获取字符串时可能用到GetDlgItemText、GetWindowText等函数。查看其IAT，如下图，发现使用的是GetDlgItemText。因此定位到GetDlgItemText的调用就可以了。

   ![图2 关键IAT](https://chrishuppor.github.io/image/Snipaste_2019-04-29_15-32-00.PNG)

3. 拖入IDA

   1. 按G键，输入“GetDlgItemTextA”，跳转到GetDlgItemTextA地址

      ![图3 jump GetDlgItemTextA](https://chrishuppor.github.io/image/Snipaste_2019-04-29_15-36-42.PNG)

   2. 按X键，查看GetDlgItemText引用情况，跳转到GetDlgItemText引用位置

      ![图4 查看交叉引用](https://chrishuppor.github.io/image/Snipaste_2019-04-29_15-38-18.PNG)

   3. 按F5，反编译该段代码

      如下图，红框中表达式如果为真，则弹出Congratulation(通过查看Text可知)；为假则弹出Incorrect Password。可见，这里红框中表达式就是比较的核心逻辑。

      ![图5 核心语句](https://chrishuppor.github.io/image/Snipaste_2019-04-29_15-39-38.PNG)

      通过GetDlgItemTextA可知，input_str就是输入的字符串。如下图，根据红线可知，v3在input_str所在地址的下一个位置，&input_str指向第一个字符，&v3指向输入字符串的第二个字符，&v4指向第三个字符，&v5指向第五个字符。比较表达式为真，则要求v3是a, v4是5，*((&v4)+1)是y，&v5指向R3versing，input_str为E——所以输入应为：Ea5yR3versing。

      ![图6 变量间地址关系](https://chrishuppor.github.io/image/Snipaste_2019-04-29_15-43-06.PNG)

## 1.2 小结

* 这个验证码匹配使用的最简单的方法——字符串比较。只不过没有将完整的字符串直接写在程序中，而是将字符串拆开进行分段比较。
* 分析时会遇到很多项v3 v4 v5这种看上去不清不楚的变量，有时只需注意他们在栈中的位置关系就可以知道他们分别代表什么。
* 由API调用入手，找到关键位置。

# 2 Easy Keygen.exe

## 2.1 破解过程

1. 了解程序功能

   输入name和serial，如果不匹配就输出wrong

   ![图7 程序界面](https://chrishuppor.github.io/image/Snipaste_2019-04-29_16-32-9.PNG)

2. 猜测关键点

   比较之后输出wrong，所以找到输出wrong的代码，其前面就是比较的逻辑。

   wrong应该是使用printf等函数输出的，可以先定位wrong字符串，然后查看其引用就可以找到输出的代码。

3. 查壳

   没壳（其实想也知道没壳）

   ![图8 PEiD查壳](https://chrishuppor.github.io/image/Snipaste_2019-04-29_16-35-54.PNG)

4. 拖进IDA

   1. 查看string，找到wrong

      ![图9 查看string](https://chrishuppor.github.io/image/Snipaste_2019-04-29_16-39-27.PNG)

   2. 查看wrong的引用

      ![图10 查看wrong引用](https://chrishuppor.github.io/image/Snipaste_2019-04-29_16-41-09.PNG)

   3. wrong唯一的引用在main函数中，main函数流程如下：

      1. 接收name输入

      2. 处理name字符串

         依次将字符与指定字符异或，并以“%02X”的形式输出到v13中

         ![图11 name处理语句](https://chrishuppor.github.io/image/Snipaste_2019-04-29_16-53-37.PNG)

      3. 接收serial

      4. 比较serial和处理后的name

         ![图12 匹配核心代码](https://chrishuppor.github.io/image/Snipaste_2019-04-29_16-54-56.PNG)

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

## 2.2 小结

* 这个验证码匹配使用的是第二阶的验证方法——比较计算结果。不进行字符串比较，而是将输入的字符串进行一系列计算，然后将计算结果与正确的值进行比较。这种方法的破解难度与获得正向算法的逆算法的难度相关。
* 由提示字符串入手，找到关键位置。

# 3 Easy Unpack.exe

这是个简单的带壳程序，会被杀软查杀，因此分析时需要关闭杀软，关闭windows defender

## 3.1 破解过程

1. 查看ReadMe

   要求找到OEP（说明这是个加壳文件）

2. 查壳

   使用PEiD查看程序，没有发现喜闻乐见的壳，看来只能自己找。

   ![图13 PEiD查壳](https://chrishuppor.github.io/image/Snipaste_2019-04-29_21-52-09.PNG)

3. 拖进OD，使用OD自解压直接获得OEP

## 3.2 小结

* 使用OD自解压功能
* 也可以自己看，这个壳十分简单，脱壳后有一个明显的JMP

# 4 Music Player

## 4.1 破解过程

1. 运行程序

   这是一个MP3播放程序，将MP3文件播放一分钟后会弹出一个提示框。Readme要求绕过“一分钟”的限制。

   猜测关键位置应该是一个cmp和je系列指令的组合——先比较时间，然后进行跳转：如果不到一分钟就继续播放，过了一分钟就进行检测，通过检测就给出Flag，不通过就弹出提示框。

   ![图14 程序界面](https://chrishuppor.github.io/image/Snipaste_2019-05-04_19-24-56.PNG)

2. 拖进PEiD

   这是个VB程序，没有加壳。

   ![图15 PEiD查壳](https://chrishuppor.github.io/image/Snipaste_2019-05-04_19-08-05.PNG)

3. 拖进VB Decompile

   如图，可以看到两个带有“TIMER”字样的函数。既然破解目的是突破时间限制，那么这很可能是关键代码所在位置。查看其它函数，发现确实没有感兴趣的地方。

   在TMR_POS_Timer中，可以看到一个与60000的比较。注意到60000ms = 60s，所以这里应该就是时间控制的跳转——如果时间大于60s，则停止播放，并弹出提示框。但这不是Flag的提示框，所以我们要控制程序直接跳到4045FE。

   ![图16 VB Decompile查看反编译代码](https://chrishuppor.github.io/image/Snipaste_2019-05-04_19-37-58.PNG)

   到此，其他地方没有看出问题，直接拖进OD吧。

4. 拖进OD

   1. 修改0x40456B的跳转，使其直接跳转到0x4045FE。删除0x40456b断点，继续运行程序，看看会发生什么。（需要忽略所有异常才能正常运行。）如图，成功突破60s的限制，但是没有弹出Flag，而是弹出来一个运行异常提示框。

      ![图17 程序异常提示框](https://chrishuppor.github.io/image/Snipaste_2019-05-04_20-12-33.PNG)

      弹出这个提示框后，点击确定，程序会退出。在程序中搜索字符串，也无法找到相关字符。经过Google得知这个提示框是windows系统弹出的，用于提示用户发生了异常。

      保存修改的程序。

   2. 如果想弹出Flag，就不能弹出这个异常，就需要知道这个异常是谁抛出的，然后绕过它。

      需要在RaiseException处下断点，运行修改程序。程序中断后，查看堆栈中函数的返回地址，依次查询直到该地址落在主模块领域。

      结果发现RaiseException的触发在0x4046B9的vbaHresultCheckObj函数调用中。

      经查询，vbaHresultCheckObj函数是一个对象检查函数，如果该对象没有通过检查则触发异常。如图，在vbaHresultCheckObj上方有一个条件跳转，如果满足条件则会跳过vbaHresultCheckObj调用，现在直接将其修改为jmp。

      ![图18 关键跳转代码](https://chrishuppor.github.io/image/Snipaste_2019-05-04_23-04-10.PNG)

      保存修改结果，运行修改后的程序。

5. Flag

   如图，Flag不是用MsgBox弹出，而是使用修改对话框标题的方式给出的。这就防止破解者通过查找MsgBox调用地址直接找到关键位置。



![图19 Flag](https://chrishuppor.github.io/image/Snipaste_2019-05-04_18-59-44.PNG)

## 4.2 小结

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

  ![图20 OD追踪函数](https://chrishuppor.github.io/image/Snipaste_2019-05-04_21-19-46.PNG)

  2. 继续运行程序，中断在MsgBox入口，此时转到栈中返回地址就可以找到一个MsgBox的调用。

     ![图21](https://chrishuppor.github.io/image/Snipaste_2019-05-04_21-25-29.PNG)

  3. 指向该调用，右键“查找引用->调用目标”就可以查看该函数的所有调用了。

# 5 Replace

## 5.1 破解过程

1. 运行程序

   程序需要输入一个字符串，然后点击check。如果失败，程序会崩溃；猜测如果成功，程序会改变下方提示信息。

2. 拖进PEiD、virustotal，没有什么异常。

3. 拖进IDA

   1. 查看字符串——有一个“correct”，应该是成功后会显示的信息。

   2. 查看IAT——有一个GetDlgItemInt函数，这个函数应该是从输入框中获取信息的。这个函数将输入框中的数据以整数的形式读取并返回。

   3. 转到correct的引用，使用F5，得到主逻辑反编译代码

      这个代码十分简单——从输入框获取数字，调用sub_404689和sub_40466F两个函数。其中，sub_404689函数就是将输入的数字加一，而sub_40466F却无法反编译。

      ![图1 check顶层代码](https://chrishuppor.github.io/image/Snipaste_2019-05-05_21-29-33.PNG)

      根据```*(_DWORD *)sub_40466F = 0xC39000C6```可知，sub_40466F函数的起始地址被写为C39000C6，即sub_40466F函数起始的机器码变为C39000C6，然后被调用。这里可能是关键点。

   4. 在反编译代码中找不到correct的信息,，这是十分奇怪的。直接查看correct相关汇编代码，该代码起始于0x401073，但没有发现有jmp到0x401073的引用，也就是说这处代码根本不会执行，也就意味着，破解的目的就是通过输入整数实现这段代码的运行。因此，输入肯定是引起了程序机器码的篡改。有两种猜想：

      * 利用输入覆盖某处机器码，使其jmp到0x401073
      * 利用输入覆盖某处机器码，使其不会jmp到其他地方，而是直接运行到0x401073

      当看到0x401071的jmp时，我认为极可能是第二种猜想，因此需要关注输入整数对程序代码的影响。

4. 拖进OD

   1. 定位到GetDlgItemInt函数的调用，在其调用的下一条指令处下断点。

   2. 运行程序，输入'123'，点击check

   3. 程序中断，查看中断之后的汇编代码。

      如图，首先将输入的整数赋值给[0x4084D0]，然后调用0x40466F。

      ![图2 输入整数赋值给[0x4084D0]](https://chrishuppor.github.io/image/Snipaste_2019-05-05_21-39-51.PNG)

      跟进0x40466F，发现该函数是对[0x4084D0]进行了加的操作。该函数结束后，[0x4084D0]变为[0x4084D0]+4+0x601605c7。

      接下来跳转到0x404690，然后会将[0x4084D0]赋给EAX，再将C39000C6写到[0x40466F]，之后会跳转到0x401071。

   4. 0x40469F之前的代码功能十分明确，直接运行程序到0x4046A9。

      此时0x40466F处的机器码已被修改，如图，sub_40466F的功能变为了将[EAX]修改为90。而90正式NOP，这是我们希望的0x401071处的代码，也就是说如果EAX = 0x401071，则能够实现程序逻辑篡改。

      ![图3 0x40466F处的机器码](https://chrishuppor.github.io/image/Snipaste_2019-05-05_22-01-22.PNG)

      然而0x401071的机器码是两个字节长度，一个NOP是不够的。查看0x4046A9后续代码，发现恰好之后会对EAX加一并再次调用0x40466F。说明这个思路是正确的，作者已经为我们篡改代码铺好了道路，只需要我们去触发。

   5. 追溯EAX值的来源——来自[0x4084D0]。而[0x4084D0]正是对输入整数处理后的结果存储地址，易得[0x4084D0] = （<input_int>+ 2 + 0x601605c7 + 2）mod 0x100000000。当输入能够使[0x4084D0] = 0x401071时，程序破解成功。所以 <input_int> = （0x401071 - 0x601605c7 - 4）mod 0x100000000。

   ## 5.2 小结

   * 出题思路：通过输入的数据覆写程序代码，从而篡改程序逻辑。
   * 该类题目一般都会有以下某个特征：
     * 需要执行的逻辑始终不被执行
     * 用户的输入可以引起数组越界
   * 本题使我想起了两个汇编知识
     * call XXX = jmp XXX，只不过正常情况下call XXX时，XXX代码后面有一个retn，但可以没有这个retn。
     * 默认DS:[EAX] = [EAX]

# 6 ImagePrc

## 6.1 解题过程

1. 拖进PEiD，virustotal没有发现异常。

2. 使用PEView查看IAT发现程序应该是使用WinMain的图形化程序。

3. 运行程序，果然是个画图的程序，点击check后会弹出wrong的提示框。没有可以输入字符串的地方，所以可能是要画什么东西。

4. 使用ResourceHacker查看程序资源，发现程序有一个奇怪的资源——大片的F和某些0组成的二进制数据，很容易使人联想到扫雷的地图和之前编写的迷宫的地图。因此猜想，要画的东西可能与这些0有关，可能是画的东西要落在这些0表示的位置。

5. 拖进IDA

   1. 查找wrong的位置

      wrong的引用出现在sub_401130函数中，查看这个函数的引用，发现这是一个WinProc函数。

   2. 分析WinProc函数

      msg == 0x111表示是系统消息(WM_COMMAND)，此时wparam == 100表示按下了check键

      除了按下check键之后的代码，其他代码都是winmain程序正常运行所需的代码，所以关键就在这里了。代码如下：

      ```c
      if ( wParam == 100 )                        // 按下Check
          {
            GetObjectA(hbm, 24, &pv);
            memset(&bmi, 0, 0x28u);
            bmi.bmiHeader.biHeight = cLines;
            bmi.bmiHeader.biWidth = v16;
            bmi.bmiHeader.biSize = 40;
            bmi.bmiHeader.biPlanes = 1;
            bmi.bmiHeader.biBitCount = 24;
            bmi.bmiHeader.biCompression = 0;
            GetDIBits(hdc, (HBITMAP)hbm, 0, cLines, 0, &bmi, 0);
            v8 = operator new(bmi.bmiHeader.biSizeImage);
            GetDIBits(hdc, (HBITMAP)hbm, 0, cLines, v8, &bmi, 0);
            v9 = FindResourceA(0, (LPCSTR)0x65, (LPCSTR)0x18);
            v10 = LoadResource(0, v9);
            v11 = LockResource(v10);
            v12 = 0;
            v13 = v8;
            v14 = v11 - (_BYTE *)v8;
            while ( *v13 == v13[v14] )
            {
              ++v12;
              ++v13;
              if ( v12 >= 90000 )
              {
                sub_401500(v8);
                return 0;
              }
            }
            MessageBoxA(hWnd, Text, Caption, 0x30u);
            sub_401500(v8);
            return 0;
          }
          return 0;
        }
      ```

      经过查询得知，GetObjectA用于获得一个图形对象，GetDIBits用于获得一个指定位图的位，然后将其作一个DIB使用的指定格式复制到一个缓冲区中，FindResourceA和LoadResource是资源加载时用的函数。while循环中显然是一个数组比较，如果比较未通过则弹出wrong提示框，如果通过则结束。

      至此，程序的逻辑就比较明了了：点击check后将用户画的图与程序自带的资源图相比较，如果比较不过就提示wrong，比较通过的就结束。那么Flag应该就是画出来的字符。

6. 构造正确的图像

   不可能通过画图找到与资源匹配的图像，直接将资源转换为图像就可以了。那么就需要知道图像的一些信息，包括大小、位数、格式。

   1. 通过上面代码中```bmi.bmiHeader.biBitCount = 24```可以知道这是一个24位的图像。

   2. 拖进OD，运行至GetDIBits，查看栈中参数可知这个图像宽200高150，是个RGB图像

      ![图1 GetDIBits](https://chrishuppor.github.io/image/Snipaste_2019-05-06_20-17.PNG)

   3. 将RH中的数据复制到文件中，使用python脚本将十六进制数转变为24位RGB图像，得到的图像上果然有Flag。

      ```python
      from PIL import Image
      
      fr = open('12.txt', 'r')
      text = fr.read()
      fr.close()
      
      rgb_str = ''
      for i in range(0, len(text)):
          if text[i] == 'F' or text[i] == '0':
              rgb_str = rgb_str + text[i]
      
      y = 200
      x = 150
      count = 0
      im = Image.new('RGB', (x, y)) #这里x y如果弄反了也显示不出来
      for i in range(0,x):
          for j in range(0,y):
              r = rgb_str[count] + rgb_str[count + 1]
              g = rgb_str[count + 2] + rgb_str[count + 3]
              b = rgb_str[count + 4] + rgb_str[count + 5]
              im.putpixel((i,j),(int(r, 16),int(g, 16),int(b, 16)))#rgb转化为像素
              count = count + 6
      im.save('1.bmp')
      ```

      *当然可以在网上搜到大佬写的writeup中的简洁代码，这里只贴了拙作*

## 6.2 小结

* 使用图像显示出Flag的想法很独特，但是难度不大，通过关键的API也很容易猜出作者意图。解题的关键就在于看出图片比较的代码。
* PIL是python用于处理图像数据的库，需要安装pillow，使用时引入PIL库。

# 7 Direct3DFPS

## 7.1 解题过程

1. 照例拖进PEiD、virustotal、PEView。发现IAT中有MessageBox，可能用于显示Flag。没有其他异常。

2. 这是一个GUI程序，拖进RH，没有发现有用信息。

3. 运行程序

   打开程序的瞬间是想拒绝的，莫非是个Misc？走几步，开几枪，大概了解程序的功能了——杀掉所有怪物，弹出Flag。

   但肯定没这么直接，否则就是电竞题了。

4. 这个程序是一个完整的FPS游戏，规模较大，有大量的代码，所以肯定不能直接上OD，先拖进IDA看看。

   1. 查看Strings

      看到两个有意思的字符串：Game Over和Game Clear。这很可能是游戏结束和游戏胜利的提示。

      ![图1 查看string](https://chrishuppor.github.io/image/Snipaste_2019-05-08_22-28-20.PNG)

   2. 追到Game Over的引用

      Game Over直接出现在WinMain函数中，让人感觉这个程序很是开门见山。

      如图，在一个if条件中显示出GameOver。联想到FPS中主角死亡时GameOver，也就是主角HP = 0时会GameOver，所以myBlood_407020一定是用于存储主角HP的。

      ![图2 GameOver引用](https://chrishuppor.github.io/image/Snipaste_2019-05-08_22-56-18.PNG)

      if ( myBlood_407020 <= 0 )之外的部分，也就是主角还活着的时候，应该是游戏正常运转所需要的各种操作，例如游戏画面更新、主角信息更新、怪物信息更新等的处理，并且在进行这些处理的过程中肯定要有游戏结束的判断，如果胜利则给出Flag。

   3. 追到GameClear的引用

      GameClear出现在一个很小的函数中，追踪这个函数的引用，发现它就出现在主角死亡判断后面，也就是上图中的	Game_clear_4039C0。

      代码如下，当满足一个条件时，会弹出一个MSGBOX。查看407028，这是一个长得很像Flag的字符串，但有些不是可见字符，肯定是加密过了，而解密肯定与游戏有关。

      其中， (signed int)result >= (signed int)&unk_40F8B4这个条件应该是在表示怪物全部死亡，循环判断*result != 1 应该表示怪物状态，如果怪物死亡就继续遍历，否则跳出循环。

      所以result从0x409194开始，每次+132，result = 0x40F8B4时结束——从第一只怪物的信息开始遍历，每个怪物信息大小为132，最后一只怪物信息存放于0x40F8B4。

      ```c++
      int *Game_clear_4039C0()
      {
        int *result; // eax
        result = GogoomaState_409194; //第一只怪物信息记录的地址
        while ( *result != 1 ) //遍历怪物信息，如果有怪物状态不是死亡就跳出循环
        {
          result += 132;//说明一个怪物信息struct大小为132
          if ( (signed int)result >= (signed int)&unk_40F8B4 )//说明unk_40F8B4应该是最后一只怪物信息地址，所以怪物信息存储在0x409194和0x40F8B4之间
          {
            MessageBoxA(hWnd, &FLAG_407028, "Game Clear!", 0x40u);
            return (int *)SendMessageA(hWnd, 2u, 0, 0);// Send Destroy
          }
        }
        return result; 
      }
      ```

   4. 寻找Flag的解密

      要解密Flag，肯定要引用加密后的字符串，也就是FLAG_407028。

      查看FLAG_407028的引用，如下，也是一个很小的函数。该函数对FLAG的修改十分明显：*((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4]。而且其他位置也没有跟多的FLAG的操作，所以这里就是要找的解密部分。

      ```c++
      int __thiscall ChangeFlag_403400(void *pVoid)
      {
        int result; // eax
        int HP_pos; // ecx
        int Gogoma_HP; // edx
      
        result = mayHitGogoomPos_403440(pVoid);
        if ( result != -1 )
        {
          HP_pos = 0x84 * result;
          Gogoma_HP = gogooma_HP_dword_409190[0x84 * result];
          if ( Gogoma_HP > 0 )
          {
            gogooma_HP_dword_409190[HP_pos] = Gogoma_HP - 2;
          }
          else
          {
            GogoomaState_DWORD_409194[HP_pos] = 0;
            *((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4];// change flag str
          }
        }
        return result;
      }
      ```

      接下来就要逐步分析*((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4]中各个变量的意义。

      * *((_BYTE *)&FLAG_407028 + result) = FLAG_407028[result]
      * byte_409184很可能是怪物信息相关的，因为之后从0x409194到0x40F8B4都是怪物信息了，所以byte_409184[HP_pos * 4]就是怪物信息中的东西。
        * 0x409184和0x409194都属于第一个怪物信息结构，但对应的成员不同——0x409184不知道代表什么，但确定0x409194对应标志怪物状态的成员。

      1. result

         result来自sub_403440返回值。

         追入这个函数，发现了一个有趣的循环和sub_4027C0函数。

         * 循环如下，这个循环计数器为v3。v3起始于0x408F90，结束于40F6B0，步长值为132，使用的v3[129]。是不是很眼熟？0x408F90 + 0x81就在0x409194附近，而0x40F6B0就在0x40F8B4附近。所以这里操作的也是怪物信息。

           ```c++
           do
             {
               if ( v3[129] && check_shot_4027C0((int)pVoid, &unk_407D84, (int)v3, (int)&v9) && v8 > (double)v9 )
               {
                 v8 = v9;
                 v7 = v1;
                 v2 = 1;
               }
               v3 += 132;
               ++v1;
             }
             while ( (signed int)v3 < (signed int)&flt_40F6B0 );
           ```

         * 进入sub_4027C0函数，就会看到一堆D3DXIntersectTri调用，并且返回值是BOOL，所以sub_4027C0很可能是用于计算位置是否匹配的。查看sub_4027C0引用，发现只出现在了sub_403440中。

         查看result的使用，除了FLAG中，result都是与0x84(132)一起使用的，说明result可能表示是第几个怪物。可以计算一下怪物个数——(0x40F8B4-0x409194)/132 = 50，正好与FLAG字符个数一致，更加确信result表示怪物编号。那么sub_403440可能就是射击位置计算，如果打中怪物则返回怪物编号，没打中就返回-1。

      2. HP_pos = 0x84 * result

         HP_pos = 0x84 * result，所以HP_pos是第result个怪物的某个信息在怪物信息数组中的偏移。

         究竟是什么信息的偏移？查看HP_pos的使用：首先获取一下，如果不为零则-2，为零则处理FLAG对应位置——是不是跟HP的处理很像？所以dword_409190[0x84 * result]就是第result个怪物的HP，当怪物死亡时会进行FLAG[result]^=byte_409184[HP_pos * 4]操作，也就是FLAG[result]^=byte_409184[0x84 * result * 4]。

         ```c++
         Gogoma_HP = gogooma_HP_dword_409190[0x84 * result];
         if ( Gogoma_HP > 0 )
         {
             gogooma_HP_dword_409190[HP_pos] = Gogoma_HP - 2;
         }
         else
         {
             GogoomaState_DWORD_409194[HP_pos] = 0;
             *((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4];// change flag str
         }
         ```

      3. byte_409184

         查看byte_409184内容，是空的，说明该部分数据需要程序动态加载，需要通过OD加载程序来获取这部分数据。

         但是程序初始化完成后，从OD获取的该部分数据是否是最终操作FLAG时使用的数据呢，也就是说这部分数据会不会随着游戏的展开被程序修改呢？比如怪物每掉一次血就修改一次。查看byte_409184的引用，只有这一个位置使用了byte_409184，所以可能是没有修改过。（*之所以说可能，是因为程序可以通过byte_409183 + 1来修改byte_409184的数据，破解程序很多时候就是尝试，对了就对了，如果错了则说明还有其他情况*）

      至此，就搞清了FLAG的解密过程，用python代码表示如下：

      ```python
      for i in range(0, 50):
      	FLAG_407028[i] = FLAG_407028[i] ^ byte_409184[i * 132 * 4]
      ```

5. 拖进OD，运行程序，把byte_409184-byte_40F8B4的数据dump出来

6. 编写脚本计算flag

   ```python
   #hex_array是一个数组，存储byte_409184-byte_40F8B4的数据
   
   flag_array = [67, 107, 102, 107, 98, 117, 108, 105, 76, 69, 92, 69,
                95, 90, 70, 28, 7, 37, 37, 41, 112, 23, 52, 57, 1, 22, 73, 76, 32, 21,
                 11, 15, 247, 235, 250, 232, 176, 253, 235, 188, 244, 204, 218, 159, 
                 245, 240, 232, 206,240, 169] #从IDA copy的FLAG_407028数组数据
   out_str = ''
   for i in range(0, 50):
       out_str = out_str + chr(hex_array[i * 132 *4] ^ flag_array[i])
   print(out_str)
   ```

## 7.2 小结

* 游戏类型的题目的Flag触发条件往往是达到一定的分数或杀光敌人。

  本题压根没有出现分数但出现了clear的字样，所以是杀光敌人的类型。

* 逆向的本质就是各种函数和变量用途的判断，这种判断往往与正向编程相联系

  * 例如：正常情况下FPS游戏失败的判断就是主角HP变为0，因此可以知道0x407020就是主角的HP。

* &是取地址符，&a表示变量a的存储地址，所以&unk_40F8B4表示变量unk_40F8B4的存储地址。

  **注意细节&unk_40F8B4和unk_40F8B4可不一样。**

* IDA也可以用于动态调试，所以本题也可以使用IDA把程序运行起来，然后使用IDAPython计算Flag。

  * 这样就可以直接获取byte_409184-byte_40F8B4的数据，不需要手动处理，脚本如下。

    ```python
    import idaapi
    
    flag = 0x1137028
    
    outstr = ''
    for i in range(0, 50):
        outstr = outstr + chr(Byte(flag +i) ^ Byte(0x1139184 +i * 132 * 4))
    print(outstr)
    ```

# 8 EasyELF

emmm...这个题大概是按分类放的...

## 8.1 解题过程

1. 拖入IDA

   查看字符串，找到了wrong。查看wrong的引用，找到了main，十分简单。

   ```c
   int __cdecl main()
   {
     write(1, "Reversing.Kr Easy ELF\n\n", 0x17u);
     input_8048434();
     if ( sub_8048451() == 1 )
       sub_80484F7();
     else
       write(1, "Wrong\n", 6u);
     return 0;
   }
   ```

   查看input_8048434()，发现是一个scanf。

   查看sub_8048451()，发现了很多比较，如下：

   ```c
   _BOOL4 sub_8048451()
   {
     if ( byte_804A021 != '1' ) //input[1] == '1'
       return 0;
     inputstr_804A020 ^= 0x34u;
     byte_804A022 ^= 0x32u;
     byte_804A023 ^= 0x88u;
     if ( byte_804A024 != 'X' ) //input[4] == 'X'
       return 0;
     if ( byte_804A025 )//input[5] == '\0',说明字符串到此结束
       return 0;
     if ( byte_804A022 != 0x7C ) //input[2] ^ 0x32u == 0x7c
       return 0;
     if ( inputstr_804A020 == 0x78 )//input[0] ^ 0x34u == 0x78
       return byte_804A023 == 0xDDu;//input[3] ^ 0x88u == 0xdd
     return 0;
   }
   ```

   没有input字符串的偏移，而是直接使用了字符地址。

   不过也不难看出，因为inputstr_804A020是接收输入字符串的指针地址，那么接下的地址当然是存储input[1]、input[2]...的。

## 8.2 小结

- 看出byte_804A021、byte_804A022这些是input[1]，input[2]就没什么了...十分友好的一

# 9 Position

## 9.1 解题过程

1. 阅读readme

   这是一个验证码程序，要求找出指定serial对应的name。

2. 运行程序

   有两个输入框，分别输入name和serial。验证失败时下方会显示wrong，猜测验证通过时会修改下方字符串。

3. 拖进PEiD、virustotal没有什么特别的。拖进RH也没有发现什么，就是正常的MFC程序，输入使用的edit控件，wrong使用static控件显示。

4. 拖进PEview，查看IAT找出程序可能使用的获得输入的API。

   mfc100u.dll中使用序号导入函数，需要使用dependency查看mfc100u.dll来一一查询。

5. 拖进OD

   1. 搜索字符串

      如图，很容易就发现了提示信息 correct和wrong。

      ![图1 搜索字符串](https://chrishuppor.github.io/image/Snipaste_2019-05-08_08-48-40.PNG)

   2. 转到wrong的引用，查看周围代码。

      如图，这是一个结构清晰的分支代码：如果验证成功就显示correct，不成功就跳转到显示wrong。

      ![图2 wrong的引用](https://chrishuppor.github.io/image/Snipaste_2019-05-08_08-50-04.PNG)

      查看jz之前的代码，发现是在call 00BA1740之后进行jz的，所以这个函数很可能是验证码比对函数。

   3. 查看函数00BA1740

      这个函数有些复杂，转到IDA尝试查看反编译代码。

6. 拖进IDA

   1. 查看sub_401740（这个就是之前的00BA1740函数，因为加载机制不一样，所以绝对地址不同，相对基址的地址都是1740）

      一言难尽，有很多奇怪的东西，又有很多数学运算，是验证码对比函数无疑了，接下来自顶向下逐块分析。

      1. 首先创建了三个CString对象。通过后续代码可知这三个对象分别用于存储name、serial和中间变量

         ```c++
         ATL::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>(&v50_name);
         v1 = 0;
         v53 = 0;
         ATL::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>(&v51_serial);
         ATL::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>(&v52_tmp);
         ```

      2. 接下来获取了一个edit的字符串

         ```c++
         CWnd::GetWindowTextW(a1_this + 304, &v50_name);
         ```

         因为一共有两个edit，所以有两个GetWindowTextW，分别获取的name和serial。究竟哪个是name，哪个是serial，可以通过获取的控件ID或对获取字符串的处理来判断。通过之后的代码可以判断出后一个是serial，那么前面这个就是name了。

         ```c++
         if ( *(_DWORD *)(v50_name - 12) == 4 )
         ```

         name的长度判断，要求name长为4。

      3. 接下来是一个循环

         这个循环看起来很大，其实主要是第一层的if内容较多，把if折叠就好看很多了。

         ```
         while ( (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) >= 'a'
                  && (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) <= 'z' )
          {                                           // name是小写字母
               if ( ++name_count_1 >= 4 ) {...}
          }
         ```

         如图，当name_count_1 < 4时就相当于：

         ```c++
         while ( (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) >= 'a'
                  && (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) <= 'z' )
          {                                           // name是小写字母
               ++name_count_1 >= 4;
          }
         ```

         当name_count_1 = 4时就会进入if结构，然后从if中直接跳出循环。

         所以这个循环的目的就是检查v50_name是不是由小写字符组成，检查的长度是4，说明name是一个由小写字母组成的长度为4的字符串。

      4. if ( ++name_count_1 >= 4 )内部首先是一段故伎重演的循环

         ```c++
         LABEL_7:
         	v4 = 0; 
         while ( 1 )
         {
         	if ( v1 != v4 )//逐个查看v50_name[v4]是否与v50_name[v1]相同
         	{
         		v5 = ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, v4);// v5 = name[v4]
         		if ( (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, v1) == v5 )
         			goto LABEL_2;                     // 要求name的四个字符互不相同
         	}
         	if ( ++v4 >= 4 )
         	{
         		if ( ++v1 < 4 )//当v4 = 4, v1 < 4时，重置v4，再比较一遍v50_name[v1]和v50_name其他字符
         			goto LABEL_7;
         		...
         	}
         	...
         }
         ```

         这段代码用于比较name中的字符，要求name的四个字符互不相同。

      5. 当name通过检查后，才会进行serial的获取

         ```c++
         CWnd::GetWindowTextW(a1_this + 420, &v51_serial);
         ```

         然后是serial的格式验证：

         ```c++
         if ( *(_DWORD *)(v51_serial - 12) == 11
                       && (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v51_serial, 5) == '-' )// 要求serial格式xxxxx-xxxxx
         ```

         要求serial是一个长为11的字符串，且serial[5] =='-'。这与readme给出的serial格式一致，也是因此判断这里是serial，上面那个是name。

      6. serial格式验证通过后就是通过name计算出结果，将结果与serial比对的过程。

         这部分有两个关键点：

         - 知道GetAt(a, pos) = a[pos]
         - 明白(a >> count )&1表示取a的第count+1位，例如 ((name_0 >> 4) & 1) 是取name_0从右向左第5个比特
         - 知道itow_s(v, tmp, 0xAu, 10)表示将v转化为十进制，并以字符串的形式存储在tmp中，tmp最大长度为0xA。
         - 小写字母的都是的ascii码都是11xxxxx，所以没有计算第6、7位，逆回去的时候也不需要计算。

         这样就能得到正向计算公式：

         ```
         name[0]0 + name[1]2 + 6 = serial[0]
         name[0]3 + name[1]3 + 6 = serial[1]
         name[0]1 + name[1]4 + 6 = serial[2]
         name[0]2 + name[1]0 + 6 = serial[3]
         name[0]4 + name[1]1 + 6 = serial[4]
         
         name[2]0 + name[3]2 + 6 = serial[6]
         name[2]3 + name[3]3 + 6 = serial[7]
         name[2]1 + name[3]4 + 6 = serial[8]
         name[2]2 + name[3]0 + 6 = serial[9]
         name[2]4 + name[3]1 + 6 = serial[10]
         ```

         已知serial，从而可以知道：

         ```c++
         name[0]0 + name[1]2 = 1 
         name[0]3 + name[1]3 = 0
         name[0]1 + name[1]4 = 2
         name[0]2 + name[1]0 = 1
         name[0]4 + name[1]1 = 0
         
         name[2]0 + name[3]2 = 1
         name[2]3 + name[3]3 = 1
         name[2]1 + name[3]4 = 1
         name[2]2 + name[3]0 = 1
         name[2]4 + name[3]1 = 0
         ```

         已知name[3] = 'p'，所以易得name[2] = 'm'
         因为单个bit只能是0或1，所以——结果为1，说明两个比特一个为0一个为1；结果为0，说明二者都为0；结果为2，说明二者都为1。由此可以推出 name[0] = 01100x1x,name[1] = 01110x0x 

         使用py计算一下可以得到四个结果，其中一个因为要求name四个字符互不相同而去掉，另外三个都可以，其中一个就是Flag。

## 9.2 所遇问题

1. **问题：**因为缺少mfc100u.dll和msvcr100.dll无法运行程序，所以先为分析机安装了这两个dll。之后又会说无法定位输入点xxx与mfc100u.dll，网上说是dll版本与机器不和，装个vs2010就可以了。

   **解决方案：**这个程序没有毒，我的主机又装有vs，所以最终决定在主机进行动态分析。

2. **问题：**拖进IDA后，发现有很多函数，根本不知道要看哪一个。MFC是事件驱动的，不像CUI程序有一个main入口，之后一切由main展开。

   **解决方案：**把程序拖入OD，通过断点或其他来定位要查看的函数地址，然后回到IDA中查看该函数。

3. **问题：**目标函数中使用了模板类，而且通过偏移地址来获取类成员的值，不容易看出这个值的用途。

   **解决方案：**额，慢慢查，耐心的看算不算解决办法...其实明白那些奇怪的偏移是在获取类成员变量的值就一切好说了，例如```if ( *(_DWORD *)(v50_name - 12) == 4 )  ```

## 9.3 小结

- 这是个KeyGen的程序，模式是通过用户输入计算出serial，本质上与第二题没有区别。之前也说了，这种程序破解的关键在于获取计算算法的逆算法，这个也是破解难度的关键。
- 需要熟悉对整数的位操作
  - tmp & 1表示取tmp的最低位，当与(tmp >> count)配合时就可以获取tmp的第(count+1)位。
- 破解MFC程序时，可以先通过OD找到关键函数，然后通过IDA分析函数的反编译代码。

# 10 Ransomware

## 10.1 解题过程

1. 查看文件

   有一个exe，一个file，一个readme.

   - 查看readme，内容如下。

     只知道要解密file，又提到exe，难道是说这个file可能跟exe有关?猜测Flag可能是解密密钥。

     ```
     Decrypt File (EXE)
     By Pyutic
     ```

   - 使用UE查看file，一个二进制文件。

2. 把run.exe拖进PEiD，有一个upx壳。

   使用upx脱壳。

3. 把run.exe拖进IDA

   1. 加载了很久——这个很奇怪，这个文件又不大*（后面知道了这个程序有很多垃圾指令，不知道是不是这个原因）*

   2. 查看string

      string很少，但还是有一个关键的字符串“key:”。

      ![图1 string列表](https://chrishuppor.github.io/image/Snipaste_2019-05-14_11-06-25.PNG)

   3. 转到key的引用，按F5，但是因为函数太大而反编译失败。

      查看附近的汇编代码，发现key引用之前有大片的无用代码，如下。

      ```
      .text:0044A763                 push    eax
      .text:0044A764                 pop     eax
      .text:0044A765                 push    ebx
      .text:0044A766                 pop     ebx
      .text:0044A767                 pusha
      .text:0044A768                 popa
      .text:0044A769                 nop
      .text:0044A76A                 push    eax
      .text:0044A76B                 pop     eax
      .text:0044A76C                 push    ebx
      .text:0044A76D                 pop     ebx
      .text:0044A76E                 pusha
      .text:0044A76F                 popa
      .text:0044A770                 nop
      .text:0044A771                 push    eax
      .text:0044A772                 pop     eax
      ```

      说明程序除了加壳，还做了混淆，使函数变得巨大，从而使IDA反编译失败。

   4. 去掉混淆

      垃圾代码不影响程序逻辑，可以直接去掉。去掉的方法是将本函数中第一条垃圾指令前的代码复制到垃圾指令后第一条有意义代码的前面。

      本题比较简单，直接从函数入口开始到key的引用都是垃圾指令，因此可以直接将函数入口代码复制到key引用之前。*（ps:当前地址属于哪个函数可以从窗口下方看到）*

      也就是将函数开头的0x40135e0到0x4135e7的二进制代码复制到0x44a76c到0x44a774。

      之后，"右键->edit function"修改main函数入口地址为0x44a76c，然后F5就可以成功了。

   5. 查看反编译后的main函数

      1. 其中有一个sub_401000函数，查看这个函数，也是一堆垃圾指令，看来也是混淆过的。使用idapython查找其有意义的指令位置，结果发现整个函数就没有用。脚本如下：

         ```python
         import idaapi
         
         ea = 0x401006
         command = ["push", "pop", "pusha", "popa", "nop"]
         while idc.GetMnem(ea) in command:
         	ea = idc.NextHead(ea)
         print(hex(ea)) #最终的输出是sub_401000的retn地址
         ```

         因此，可以将程序中所有的call sub_401000去掉。脚本如下：

         ```python
         import idaapi,ida_bytes
         
         ea = 0x44A770
         ed = 0x44a983
         x = 0x90
         while ea < ed:
             if idc.GetDisasm(ea) == 'call    sub_401000':
                 for i in range(0, 5):
                     ida_bytes.put_bytes(ea + i, chr(x))
             ea = idc.NextHead(ea)
         ```

      2. 此时main函数就变得十分清晰了——接收一个输入，打开文件并读取文件信息，将读取的信息进行变换，将变换后的数据写入原文件中。

         代码如下

         ```c
           //接收输入
           printf("Key : ");
           scanf("%s", byte_44D370);
           v5 = strlen(byte_44D370);
           v6 = 0;
           File = fopen("file", "rb");
           if ( !File )
           {
             printf(asc_44C1C4);
             exit(0);
           }
           fseek(File, 0, 2);
           ftell(File);
           rewind(File);
           while ( !feof(File) )
             byte_5415B8[v6++] = fgetc(File);
           for ( i = 0; i < v4; ++i )
           {
             byte_5415B8[i] ^= byte_44D370[i % v5];
             byte_5415B8[i] = ~byte_5415B8[i];
           }
           fclose(File);
           v3 = fopen("file", "wb");
           for ( j = 0; j < v4; ++j )
           {
             savedregs = v3;
             fputc(byte_5415B8[j], v3);
           }
           printf(asc_44C1E8);
         ```

         如下，解密操作即文件数据与输入信息异或，然后自身取反。

         ```
           for ( i = 0; i < v4; ++i )
           {
             byte_5415B8[i] ^= byte_44D370[i % v5];
             byte_5415B8[i] = ~byte_5415B8[i];
           }
         ```

         所以输入的字符=（密文取反）^(明文)。

4. 接下来就是通过密文和明文求输入字符。

   密文可以直接从file中获取，明文去哪里找？根据readme提示，file是一个exe文件，而所有的exe都在固定的位置有一个字符串“This program cannot be run in DOS mode”，所以可以通过这段明文求解，其密文即为file中对应位置的字节。

   解密代码如下：

   ```python
   m = "This program cannot be run in DOS mode"
   c = [0xC7, 0xF2, 0xE2, 0xFF, 0xAF, 0xE3, 0xEC, 0xE9, 0xFB, 0xE5, 0xFB, 0xE1, 0xAC, 0xF0, 0xFB, 0xE5, 0xE2, 0xE0, 0xE7, 0xBE, 0xE4, 0xF9, 0xB7, 0xE8, 0xF9, 0xE2, 0xB3, 0xF3, 0xE5, 0xAC, 0xCB, 0xDC, 0xCD, 0xA6, 0xF1, 0xF8, 0xFE, 0xe9]
   rc = ''
   for i in range(0, 38):
       rc = rc + chr(ord(m[i])^((~c[i])%0x100)) #注意这里要%0x100，否则~c[i]大小会超过1B
   print (rc)
   ```

   将密钥输入验证，发现居然不是Flag。

5. 既然file是一个exe，那么密钥可能是这个程序给出的。

   运行run.exe，输入密钥，得到解密的file。修改file后缀，运行file.exe。输出了一个字符串，根据提示信息，这个就是Flag了。*(作者的Flag不叫flag，而是key，我又孤陋寡闻了...)*

## 10.2 小结

- 刚开始看readme时会一头雾水，因为把Decrypt File理解成了“解密某个文件”，而实际上这里的File是文件中的file的文件名。*（如果是解密某个文件，应该为Decrypt a File。英语还得学呀，要不然注意不到这些细节的差别。）*
- 代码混淆的一种——程序中含有大量的垃圾代码
  - 当函数中含有大量垃圾代码时，函数会变得特别大，进而IDA反编译失败。此时需要越过垃圾代码，将函数入口的机器码复制到函数有意义的代码前面，调整函数起始地址，然后就可以反编译了。
  - 当整个函数都是垃圾代码时，可以将该函数的所有调用从程序中删除。

# 11 Twist1

题如其名，很考研动态调试的一道题，OD用的熟不熟很关键。

## 11.1 解题过程

1. 拖入PEiD和virustotal没有发现异常。

2. 拖入PEview，查看导入表，发现只导入了kernel32的函数，有些奇怪。

3. 根据程序名称猜测这个程序很可能使用了某些变换，使得程序难以调试，因此打算使用promon和proexp监控一下。

   1. 运行程序，使用processexplorer查看程序进程，发现磁盘映像中的字符串与内存中的字符串大不相同，所以这个程序很可能加了壳，而且是PEiD识别不了的壳。
   2. 打开promon监控，重新运行程序，没有发现异常行为。

4. 拖入IDA，函数列表十分诡异，验证了加壳的猜想。

5. 拖入OD

   1. 尝试直接运行程序，使其停在等待用户输入的地方，失败。

   2. 尝试使用多种断点方法获取OEP，均失败。

   3. 看来只能自己单步调了

      1. 看到了[FS:30]以及对其偏移的操作，反调实锤了。

         最终EDX = EAX + 0X18，而eax == [FS:30]，所以这里用PEB.ProcessHeap来反调。

         ![图1 检查PEB.ProcessHeap代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-26-12.PNG)

         如下图，检查PEB.ProcessHeap.Flag是否为2，也就是检查[[FS:30] + 0x18]+0xC是否是2。（非调试时应该不为2）

         ![图2 检查PEB.ProcessHeap.Flag代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-27-53.PNG)

         接下来又检查了[[FS:30] + 0x18]+0x40是否为2。（非调试时应该不为2）

         ![图3 检查FS:30 + 0x18+0x40代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-32-47.PNG)

         接下来再次获取了PEB.ProcessHeap，并检查[[FS:30] + 0x18]+0x44是否为0.（非调试时不为0）

         ![图4 ((FS:30) + 0x18)+0x44)检查代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-34-34.PNG)

         接下来惊现与0xABABABAB和0xEEFEEEFE的比较。这个是通过堆中数据检查进行反调中常用的比较。查看EDI的来源，果然是来自于堆空间。此时需要将所检查的空间中的0xABABABAB和0xEEFEEEFE改为其他数据才能通过测试，或者修改跳转逻辑。

         ![图5 堆空间检查代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-36-55.PNG)

         接下来发现注册了一个SEH，将函数403903注册为异常处理函数。

         ![图6 SEH注册代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-47-57.PNG)

         接下来看到了GetVersion和GetCommand，是不是见到亲人了？没错，这里就是start函数了，之前注册SEH也是start的正常操作。

         ![图7 OEP代码](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-48-55.PNG)

         接着根据main函数的参数特点结合个人经验就找到main函数入口了，如下。

         ![图8 main](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-54-44.PNG)

      2. 至此，程序已经恢复了无壳状态，可以dump下来然后修复了。

         尝试运行修复后的无壳exe，失败。

      3. 继续单步调试，追入main函数

         1. 一开始居然是一个垃圾函数，真是开门“喜”
         2. 步入401000函数
            1. 又见FS:30，不过这次不是反调，而是读取了Ldr中的dll链表.
         3. 接下来是401090函数，这个函数是自定义的GetProcAddress
         4. ......好多自定义函数，不过都无所谓......
         5. 最后找到401240，这是correct提示信息前的最后一个函数了，说明这个就是关键函数了。

      4. 步入401240

         1. 将输入copy到了一个位置

         2. 使用了NtQueryInformationProcess，但不是直接用的，而是通过409150调用。

            1. 先检查NtQueryInformationProcess函数起始机器码是否正确，也就是检查这个函数是否被Hook。
            2. 如果通过检查，则将第二个参数设为7，也就是ProcessDebugPort，这又是一种反调。需要将返回值修改为0。
            3. 接着使用了 0x1E(ProcessDebugObjectHandle)来获取调试对象。但在检查时没有检查返回值，而是检查了系统异常。非调试状态下，函数返回NULL，说明没有找到调试对象，系统应该报出c0000353的异常。
            4. 接着检查了0x1F(ProcessDebugFlag)，调试状态下会返回0。

         3. 然后将input[0]存入0x40B990，把input[6]存入了0x40B991，并将input[6]^0x36存入了0x40C450。

         4. 接着通过GetThreadContext函数获取寄存器的值，然后检测硬件断点寄存器。

            目前，我还不知道具体原理，但是查看这部分jne，都是跳转到反调测试未通过的部分，因此只需要把jne改为nop或修改符号寄存器就可以了。

         5. 接着就是梦寐以求的字符串匹配部分了。

            这个部分也不太平，很考验耐心和细心。而且各种跳转，所以就不截图了。*(也许这里应该用pin来做)*

            1. 检查0x40C450是否为0x36，也就是说input[6]^0x36==0x36，所以input[6] == 0

            2. 将Input打乱顺序存储到内存，这些内存地址就是接下来要使用的东西，注意记录对应关系就好，不再细说。

               1. 检查[0x40B000]是否为0x49，而[0x40B000] == input[0] ror 6，所以input[0] == 0x49 rol 6 == 'R'

               2. 检查[0x40CCE0] ^ 0x77是否为0x35，所以input[2] == 'B'

               3. 检查[0x40CCEC]^0x20是否为0x69，所以input[1] == 'I'

               4. 接下来居然又检查了一下input[0]，而且跟第一次的值还不一样。

                  用户的输入只有一个值，在这两次不同的检查之间也没有进行修改，而且确信第一次检查结果要为真，所以这次检查结果为假。这是一个用来迷惑破解者的小把戏。

               5. 检查[0x40C401]^0x21是否为0x64，所以input[3] == 'E'

               6. 检查[0x40CD30]^0x46是否为0x8。很早之前有一个将input[4]写入0x40CD30的操作，但是如果没有记录的话也可以尝试查看0x40CD30。查看0x40CD30发现是我input的第5个字符，所以应该是input[4]。因此，input[4] == 'N' 

               7. [0x40CFFC] rol 4 == 0x14，所以input[5] == 0x14 ror 4 == 0x65 == 'A'。

## 11.2 小结

- 这道题一共有三个点，每个点都有一定难度，稍有不慎就要重来。
  - 脱壳找到OEP
  - 绕过所有反调
  - 破解input匹配逻辑
- 直接运行程序是可以的，使用OD加载运行程序失败的——程序极可能有**反调功能**，反调总结请见另一片博客 [静态反调小结]
- 单步调试时注意不要步过call，而是要追进去，要不然哪里崩的都不知道......
- **OD的标签**：如果某个函数将程序导向错误的路径，要使用shift+;为其添加标签，这样在条件跳转时就能清楚的知道如何跳转才是正确的。

*PS:被这个题的反调虐哭，几乎使用了全部的静态反调手段，是个专场......其中大部分在《逆向工程核心原理》里面有讲解。其中，检测HEAP.0x40和HEAP.0x44的值，检测关键函数是否被挂钩，以及使用GetThreadContext检测硬件调试寄存器的方法是我新学到的。真心的感谢作者。*

# 12 WindowKernel

是一个与驱动有关的题目，如果对驱动比较熟悉就是它左边那个题的难度，否则就是它上边那个题的难度。

## 12.1 解题过程

1. 运行程序

   1. 拖进64位机器，运行exe，提示无法创建服务。以管理员权限运行，提示无法启动服务。很明显，服务程序是sys，如果服务启动失败就是sys的安装问题。
   2. sys安装失败，有可能是位数不合适。查看sys位数，果然是32位。
   3. 拖进32位机器，以管理员权限运行exe，显示出提示信息"Analyze me! :>Hit)Keyboard"。
   4. 下方是一个不可输入的文字框，旁边有一个enable的按钮。点击按钮，发现按钮文字变成了check，文字框也变得可以输入了。输入几个字符，点击按钮，弹出提示框wrong，并恢复到enable状态。

2. 将exe拖入IDA

   1. 窗口函数是自定义的DialogFunc。

      通过查询可知DialogFunc参数宏定义在WinUser.h中，查看其宏定义可知，272表示WM_INITDIALOG，273表示WM_COMMAND，275表示WM_TIMER，a3 == 2表示WM_DESTROY 。没有看到1002和1003相关，可能是自定义的消息。

      ```c
      BOOL __stdcall DialogFunc(HWND hWnd, UINT a2, WPARAM a3, LPARAM a4)
      {
        if ( a2 == 272 )//WM_INITDIALOG
        {
          SetDlgItemTextW(hWnd, 1001, L"Wait.. ");
          SetTimer(hWnd, 0x464u, 0x3E8u, 0);
          return 1;
        }
        if ( a2 != 273 )//not wm_command
        {
          if ( a2 == 275 )// WM_TIMER
          {
            KillTimer(hWnd, 0x464u);
            sub_401310(hWnd);
            return 1;
          }
          return 0;
        }
        if ( (unsigned __int16)a3 == 2 ) //WM_DESTROY 
        {
          SetDlgItemTextW(hWnd, 1001, L"Wait.. ");
          sub_401490();
          EndDialog(hWnd, 2);
          return 1;
        }
        if ( (unsigned __int16)a3 == 1002 )
        {
          if ( a3 >> 16 == 1024 )
          {
            Sleep(0x1F4u);
            return 1;
          }
          return 1;
        }
        if ( (unsigned __int16)a3 != 1003 )
          return 0;
        sub_401110(hWnd);
        return 1;
      }
      ```

      DialogFunc逻辑很清晰：创建对话框的时候添加一个定时器，当超时时创建并启动服务程序，服务由WinKer.sys来提供。程序结束时关闭服务程序。当a3位1003时，调用sub_401110函数。

   2. 查看sub_401110

      如下，发现了最后的提示信息```MessageBoxW(v1, L"Correct!", L"Reversing.Kr", 0x40u);```，说明这个函数就是check相关核心函数。

      ```c
      HWND __thiscall sub_401110(HWND hDlg)
      {
        v1 = hDlg;
        GetDlgItemTextW(hDlg, 1003, &String, 512);
        if ( lstrcmpW(&String, L"Enable") )
        {
          result = (HWND)lstrcmpW(&String, L"Check");
          if ( !result )
          {
            if ( sub_401280(0x2000) == 1 )
              MessageBoxW(v1, L"Correct!", L"Reversing.Kr", 0x40u);
            else
              MessageBoxW(v1, L"Wrong", L"Reversing.Kr", 0x10u);
            SetDlgItemTextW(v1, 1002, &word_4021F0);
            v5 = GetDlgItem(v1, 1002);
            EnableWindow(v5, 0);
            result = (HWND)SetDlgItemTextW(v1, 1003, L"Enable");
          }
        }
        else if ( sub_401280(4096) )
        {
          v3 = GetDlgItem(v1, 1002);
          EnableWindow(v3, 1);
          SetDlgItemTextW(v1, 1003, L"Check");
          SetDlgItemTextW(v1, 1002, &word_4021F0);
          v4 = GetDlgItem(v1, 1002);
          result = SetFocus(v4);
        }
        else
        {
          result = (HWND)MessageBoxW(v1, L"Device Error", L"Reversing.Kr", 0x10u);
        }
        return result;
      }
      ```

      1. 如图，使用RH查看exe资源文件，1003正是按钮的ID。

         ![图1 Dialog资源ID查看](https://chrishuppor.github.io/image/Snipaste_2019-05-21_09-15-14.PNG)

         所以```GetDlgItemTextW(hDlg, 1003, &String, 512);```获取的是按钮显示的文字。

      2. 如下，如果按钮是check且sub_401280(0x2000) == 1，则表示验证通过，否则恢复enable状态。

         ```c
         if ( lstrcmpW(&String, L"Enable") )
         {
             result = (HWND)lstrcmpW(&String, L"Check");
             if ( !result )
             {
                 if ( sub_401280(0x2000) == 1 )
                     MessageBoxW(v1, L"Correct!", L"Reversing.Kr", 0x40u);
                 else
                     MessageBoxW(v1, L"Wrong", L"Reversing.Kr", 0x10u);
                 SetDlgItemTextW(v1, 1002, &word_4021F0);
                 v5 = GetDlgItem(v1, 1002);
                 EnableWindow(v5, 0);
                 result = (HWND)SetDlgItemTextW(v1, 1003, L"Enable");
             }
         }
         ```

      3. 如下，如果sub_401280(0x1000)为真则变为check状态。

         ```c
         else if ( sub_401280(0x1000) )
         {
             v3 = GetDlgItem(v1, 1002);
             EnableWindow(v3, 1);
             SetDlgItemTextW(v1, 1003, L"Check");
             SetDlgItemTextW(v1, 1002, &word_4021F0);
             v4 = GetDlgItem(v1, 1002);
             result = SetFocus(v4);
         }
         ```

      4. 由此可见，sub_401280十分关键，当参数为0x1000时控制exe变换状态，当参数为0x2000时给出输入是否正确的判断。

         如下，可知0x1000和0x2000的参数名为dwIoControlCode，会进一步传递给DeviceIoControl函数。DeviceIoControl返回数据存储在OutBuffer中，并作为sub_401280返回值。

         ```c
         int __usercall sub_401280@<eax>(HWND a1@<edi>, DWORD dwIoControlCode)
         {
             v2 = CreateFileW(L"\\\\.\\RevKr", 0xC0000000, 0, 0, 3u, 0, 0);
             if ( v2 == (HANDLE)-1 )
             {
                 MessageBoxW(a1, L"[Error] CreateFile", L"Reversing.Kr", 0x10u);
                 result = 0;
             }
             else if ( DeviceIoControl(v2, dwIoControlCode, 0, 0, &OutBuffer, 4u, &BytesReturned, 0) )
             {
                 CloseHandle(v2);
                 result = OutBuffer;
             }
             else
             {
                 MessageBoxW(a1, L"[Error] DeviceIoControl", L"Reversing.Kr", 0x10u);
                 result = 0;
             }
             return result;
         }
         ```

         通过查询可知，DeviceIoControl函数用于“Sends a control code directly to a specified device driver, causing the corresponding device to perform the corresponding operation.”。

         经查询可知，dwIoControlCode是硬件操作码，可以自定义，由```#define CTL_CODE(DeviceType, Function, Method, Access) (((DeviceType) << 16) | ((Access) << 14) | ((Function) << 2) | (Method))```来定义一个硬件操作码，对应的宏定义在Windev.h中。但是搜了半天也没找到Windev.h，结果找到了WinIoCtl.h，这里面也有对应的宏定义。从而可知0x1000和0x2000只有Function字段有值，且function是作者自定义的。

         至此，题目的结构就清晰了——使用exe将sys注册为服务并启动，定义了0x1000和0x2000两个硬件操作码，使用DeviceIoControl函数控制驱动接收用户输入并进行比较。

   3. 动态调试sys需要配置双机调试环境，比较麻烦，因此先尝试使用IDA静态调试。将sys拖入IDA。

      1. 查看string，没有什么特别的，只好从DriveEntry看起。

         入口一共两个函数：sub_14005和sub_11466。其中，sub_14005显然跟主功能没有关系。经查询， BugCheckParameter2与系统蓝屏有关，那么就与我们要找的无关咯。所以关键在sub_11466中

      2. 查看sub_11466

         看样子像一个init函数，负责各个全局参数的初始化以及事件与函数的绑定。

         对比一个简单的驱动编写程序可知之前的猜测是正确的。如下，一共有五处函数绑定。

         ```c
         DriverObject->DriverUnload = (PDRIVER_UNLOAD)sub_1131C;
         
         DriverObject->MajorFunction[14] = (PDRIVER_DISPATCH)sub_11288;
         DriverObject->MajorFunction[0] = (PDRIVER_DISPATCH)sub_112F8;
         DriverObject->MajorFunction[2] = (PDRIVER_DISPATCH)sub_112F8;
         KeInitializeDpc(&DeviceObject->Dpc, sub_11266, DeviceObject);
         ```

         通过查询DriverObject可知，这是一个结构体，用于描述驱动运行所需要的一些数据。其成员DriverUnload用于存放"The entry pointer for the driver's Unload routine"；MajorFunction是一个数组，用于为派遣动作设定对应的派遣函数，其中MajorFunction的下标为代表IRP major function code的IRP_MJ_XXX样式的宏，定义在wdm.h中。查询DeviceObject->Dpc可知，这是一个设备对象的延迟过程调用（DPC）对象，内部以一个灵活的定时器，用于处理设备的异步操作。

         网上搜索驱动开发用到的派遣函数序号(wdm.h)，可知MajorFunction[14]表示IRP_MJ_DEVICE_CONTROL，MajorFunction[0]表示IRP_MJ_CREATE，MajorFunction[2]表示IRP_MJ_CLOSE。

         因此，sub_1131C是用于驱动卸载的，sub_112F8是用于初始化和关闭的，所以都不是我们关心的。sub_11288是设备控制函数，是分析的重点。sub_11266是设备异步操作函数，也是我们要关注的。

      3. 查看sub_11288

         尽管不知道Irp->Tail.Overlay.PacketType + 12是什么，但根据0x1000和0x2000可以猜到这个就是dwIoControlCode。当控制码为0x1000时进行初始化，当控制码为0x2000时的设置表明dword_13030很可能是一个flag，*(_DWORD *)&v3->Type很可能是返回值。

         IofCompleteRequest用于消息传递，类似于DialogFunc中的Msg传递函数。

         因此，本函数的功能是根据控制码进行配置IRP结构，然后进行消息传递。

         ```c
         int __stdcall sub_11288(int a1, PIRP Irp)
         {
             int v2; // edx@1
             struct _IRP *v3; // eax@1
         
             v2 = *(_DWORD *)(Irp->Tail.Overlay.PacketType + 12);
             v3 = Irp->AssociatedIrp.MasterIrp;
             if ( v2 == 0x1000 )
             {
                 *(_DWORD *)&v3->Type = 1;
                 dword_13030 = 1;
                 dword_13034 = 0;
                 dword_13024 = 0;
                 dword_1300C = 0;
             }
             else if ( v2 == 0x2000 )
             {
                 dword_13030 = 0;
                 *(_DWORD *)&v3->Type = dword_13024;
             }
             Irp->IoStatus.Status = 0;
             Irp->IoStatus.Information = 4;
             IofCompleteRequest(Irp, 0);
             return 0;
         }
         ```

         既然进行了消息传递，那么肯定有消息处理函数。是不是想到了之前的Dpc函数sub_11266？这个函数大概就是用于进行消息处理的——根据不同的消息进行不同的设备操作。

      4. 查看sub_11266

         本函数首先使用READ_PORT_UCHAR指定从0x60端口读取一字节数据，然后调用 sub_111DC函数处理这一字节的数据。其中，READ_PORT_UCHAR的功能很好查，通过搜索windows设备端口0x60得知，这个端口表示键盘。

         因此，本函数的功能就是从键盘读取一个字节，然后处理它，处理函数为 sub_111DC。

      5. 查看 sub_111DC

         如下，首先判断dword_1300C是否是1，如果不是1则会判断dword_13034的值，根据dword_13034值的不同有不同的操作，总结来说就是dword_13034为偶数时仅++，对a1不做处理；dword_13034为奇数时要对a1进行比较，如果不通过则将dword_1300C置为1。如果通过比较，当dword_13034为1、3、5时直接++，当dword_13034为7时则加100。如果没有dword_13034对应的值，则使用sub_11156来处理a1。

         ```c
         signed int __stdcall sub_111DC(char a1)
         {
             result = 1;
             if ( dword_1300C != 1 )
             {
                 switch ( dword_13034 )
                 {
                     case 0:
                     case 2:
                     case 4:
                     case 6:
                         goto LABEL_3;
                     case 1:
                         v2 = a1 == 0xA5u;
                         goto LABEL_6;
                     case 3:
                         v2 = a1 == 0x92u;
                         goto LABEL_6;
                     case 5:
                         v2 = a1 == 0x95u;
         LABEL_6:
                         if ( !v2 )
                             goto LABEL_7;
         LABEL_3:
                         ++dword_13034;
                         break;
                     case 7:
                         if ( a1 == 0xB0u )
                             dword_13034 = 100;
                         else
         LABEL_7:
                         dword_1300C = 1;
                         break;
                     default:
                         result = sub_11156(a1);
                         break;
                 }
             }
             return result;
         }
         ```

      6. 查看sub_11156

         发现其逻辑与sub_111DC一致，只不过在比较前对a1进行了异或——a1 = a1 ^ 0x12。如果没有dword_13034对应的值，则使用sub_110D0来处理a1。

      7. 查看sub_110D0

         发现其逻辑与sub_111DC和sub_11156一致，只不过在比较前对a1进行了异或——a1 = a1 ^ 0x5。如果没有dword_13034对应的值，则返回dword_13034-200。

      8. 将sub_111DC、sub_11156、sub_110D0连起来看就可以发现，这三个函数接力进行输入字符匹配，dword_13034则类似于一个计数器，高位用于记录是第几个处理函数，低位记录是本函数处理的第几个字符。

         之所以只处理奇数，是因为一次按键会产生两个字符，第一个字符表示按下去的按键，第二个字符表示弹起的键。虽然都是一个键，但硬件处理时要标记是弹起还是按下，所以有两个值，且按下的值在前。但是实际匹配时只需要匹配一个就行了，本题选择匹配弹起的键的值。

         因此，一共有12个字符，如果都匹配则将retn_13024设为1，否则设为0。

         根据匹配逻辑可知这十二个字符如下，注意第三个函数接收到的参数已经经过0x12异或了。

         ```python
         f = [0xA5, 0x92, 0x95, 0xb0,
              0x12^0xB2, 0x12^0x85, 0x12^0xA3, 0x12^0x86,
               0x12^0xB4^0x5, 0x12^0x5^0x8F, 0x12^0x5^0x8F, 0x12^0x5^ 0xB2]
         ```

      9. 因为这十二个字符是通过端口从键盘IO中读取的，所以这些不是ascii，而是按键的扫描码，而且是按键弹起的扫描码。需要自行对应ascii码，然后将ascii数值转换为chr.

         脚本如下：

         ```python
         ScanCode = [0X90, 0X91, 0X92, 0X93, 0X94, 0X95, 0X96, 0X97, 0X98, 0X99,
                     0x9e, 0x9f, 0xa0, 0xa1, 0xa2, 0xa3, 0xa4, 0xa5, 0xa6,
                     0xac, 0xad, 0xae, 0xaf, 0xb0, 0xb1, 0xb2]
         scanChr = 'qwertyuiopasdfghjklzxcvbnm'
         
         f = [0xA5, 0x92, 0x95, 0xb0,
              0x12^0xB2, 0x12^0x85, 0x12^0xA3, 0x12^0x86,
               0x12^0xB4^0x5, 0x12^0x5^0x8F, 0x12^0x5^0x8F, 0x12^0x5^ 0xB2]
         flagStr = ''
         for item in f:
             flagStr += (scanChr[ScanCode.index(item)])
         
         print(flagStr)
         ```

## 12.2 小结

就像开头说的一样，做题人如果熟悉甚至仅仅是编写过驱动，那么就会觉得这个题和EasyELF一样，纯靠读IDA代码就能破解。如果一点不了解驱动也没关系，只要耐心的在网上搜索，最终也能破解。

- 在使用IDA查看函数反编译代码时，一些参数常常直接显示数值而不是宏名称，这时候可以先查这个函数的参数，一般MSDN都会给出函数所在头文件，然后查看对应的头文件，从中找到值对应的宏。
- [键盘扫描码（他人博文）](https://blog.csdn.net/qq_37232329/article/details/79926440)

# 13 Reversing.kr_AutoHotKey

一个需要静态分析配合动态调试的题目——动态调试方便看内存数据，断点功能也方便根据功能定位函数的调用；静态分析方便看程序逻辑。

## 13.1 解题过程

1. 阅读Readme，需要找到一个DecryptKey的MD5和exe的MD5。

2. 拖进PEview，发现了UPX壳，脱之。将脱壳后的程序拖进PEview查看IAT，好多函数，不好说哪个比较可疑。

3. 运行脱壳后的程序，提示“EXE corrupted”；运行原程序，没有问题。说明程序有自我保护机制。

4. 将脱壳后的程序拖入IDA

   1. 查看String，查看EXE corrupted的引用

      如下，根据4508C7的返回值确定程序是否被脱壳，所以4508C7函数就是程序的完整性检验函数。

      ```c
      if ( checkfile_4508C7((FILE **)&v5, (int)lpCaption, (char *)&h) )
      {
          msg_43C205("EXE corrupted", 0, (char *)lpCaption, 0.0, 0);
      }
      ```

   2. 查看checkfile_4508C7函数

      一个比较复杂的函数，简单逻辑如下。

      ```c
      int __thiscall checkfile_4508C7(FILE **this, int a2, char *a3)
      {
          ffp = this;
          //1. 初始化v17数据
          sub_450F56(&v17);
          //2. 打开自身文件
          GetModuleFileNameA(0, &Filename, 0x104u);
          this_fp = fopen(&Filename, aRb);
          *ffp = this_fp;
          if ( !this_fp )
              return 1;
          v6 = 0;
          //3. 读取全局数据
          do
          {
              hardcode[v6] = byte_466514[v6];
              hardcode_2[v6] = byte_46650C[v6];
              ++v6;
          }
          while ( v6 < 8 );
          //4. 从文件中读取数据并进行处理
          v7 = strlen(a3);
          ffp[68] = 0;
          for ( i = 0; i < v7; ++i )
              ffp[68] = (FILE *)((char *)ffp[68] + a3[i]);
          fseek(*ffp, -8, 2);
          fread(ffp + 1, 4u, 1u, *ffp);
          v9 = ftell(*ffp);
          fp = *ffp;
          begin_offset = v9;
          fread(&v23, 4u, 1u, fp);
          fseek(*ffp, 0, 0);
          count_1 = 0;
          v11 = 0;
          if ( begin_offset > 0 )
          {
              do
              {
                  if ( (*ffp)->_flag & 0x10 )
                      break;
                  fread(&v22, 1u, 1u, *ffp);
                  v11 = sub_450F95(&v17, v22);
                  ++count_1;
              }
              while ( (signed int)count_1 < begin_offset );
          }
          if ( v23 != (v11 ^ 0xAAAAAAAA) )
              goto LABEL_25;
          fseek(*ffp, (int)ffp[1], 0);
          if ( !fread(v19, 0x10u, 1u, *ffp) )
              goto LABEL_25;
          v12 = 0;
          do
          {
              if ( v19[v12] != hardcode[v12] )
                  break;
              ++v12;
          }
          while ( v12 < 0x10 );
          if ( v12 != 0x10 )
          {
              LABEL_25:
              v16 = 3;
              LABEL_19:
              v13 = v16;
              fclose(*ffp);
              return v13;
          }
          fread(&v26, 1u, 1u, *ffp);
          if ( v26 != 3 )
          {
              v16 = 4;
              goto LABEL_19;
          }
          fread(&v24, 4u, 1u, *ffp);
          v14 = v24 ^ 0xFAC1;
          fread(ffp + 3, 1u, v24 ^ 0xFAC1, *ffp);
          //6. 计算一个数值
          sub_450ABA((int)(ffp + 3), v14, v14 + 50130);
          *((_BYTE *)ffp + v14 + 12) = 0;
          v15 = 0;
          for ( ffp[68] = 0; v15 < v14; ++v15 )
              ffp[68] = (FILE *)((char *)ffp[68] + *((char *)ffp + v15 + 12));
          ffp[2] = (FILE *)ftell(*ffp);
          return 0;
      }
      ```

      如图，只有在跳转到LABEL_25或LABEL_19才能返回非0值，而在fread(&v24, 4u, 1u, *ffp);之后就确定会返回0了，说明这一部分与校验无关，但与程序正确运行相关。也就是说校验的数据传递给sub_450ABA函数，只有校验通过才能保证sub_450ABA计算正确，猜测sub_450ABA计算的就是exe的MD5。

      这里的运算较为复杂，按照reversing.kr的习惯，应该不需要进行复杂计算，因此可能可以直接从内存中获取数据。

   3. 接下来试着从DialogFunc开始分析。啊，多么复杂的程序，根本找不到check输入的地方。

5. 将原程序拖入OD

   1. 因为之前在IAT中看到了GetWindowTextA，而且从EDIT中获取字符串的函数仅此一个，所以在这个函数下断点一定可以定位到字符串check的部分。

      ![图1 GetWindowTextA断点](https://chrishuppor.github.io/image/Snipaste_2019-05-21_22-20-29.PNG)

   2. 在GetWindowTextA处下断点，运行程序至返回。

      ![图2 返回用户模块](https://chrishuppor.github.io/image/Snipaste_2019-05-21_22-21-09.PNG)

   3. 在存储输入字符串的内存地址下内存断点，运行程序。程序中断在对输入数据的操作地址0x457A00，接下来将输入数据与[ECX]进行比较。此时发现，[ECX]中存储了一个32字节的字符串。所以这里很可能就是另一个MD5了。

6. 将两个MD5解密，分别得到pawn和isolated。

## 13.2 小结

- 逆向题毕竟不是实践，所以其中的算法一般不会很复杂，都是可解的。如果一个算法十分复杂，就要考虑是否可以通过动态运行，利用程序自身进行运算，然后从内存中提取结果。
- 动静结合分析，通过静态分析找逻辑，通过动态分析找位置。找到位置后可以返回静态进行分析。
- 通过查看他人writeup，发现AutoHotKey是一个成熟的开源框架，用于将脚本转换为exe，也可以再从exe将脚本提取出来。在转换时需要一个Password，也就是本题中的DecryptKey。如果要从exe中恢复脚本，当然要求exe不能有变化，所以会需要进行exe的md5验证，只不过作者没有直接计算exe的md5，而是自己设计了一个exe完整性校验算法。如此一来，这个题目的背景就清楚了——为从exe中恢复脚本，需保证程序完整性以及获得转换密钥。
- md5在线解密网址：[http://pmd5.com/](http://pmd5.com/)

# 14 CSHOP

是一个简单但巧妙的题，利用了“白纸白字”的思想。

## 14.1 解题过程

1. 解压程序后，没有readme。直接运行程序，发现除了一个框什么也没有。联想到之前的ImageProc，试着在框中画几笔，没有成功，果然不是一个思路。*（PS.目前还没有发现这个网站有思路重复的题目）*

2. 拖入PEiD，发现是C#程序，没有壳。

3. 拖入dnSpy，发现这个程序进行了混淆。关闭dnSpy，使用de4dot进行去混淆。

4. 将去混淆的程序拖入dnSpy。程序十分简单，一共也才不到二百行代码。

   1. 有一个派生子Form的FrmMain类，该类有四个方法

      1. InitializeCompenont：初始化函数，生成一个按钮和十个静态Label，使用Form1_Load将按钮和label的文字全部初始化为""，按钮事件由btnStart_Click处理。

         其中，为每个控件指派了TabIndex，说明可以使用Tab键操作控件。

         ```c#
         private void InitializeComponent()
         {
         	ComponentResourceManager componentResourceManager = new ComponentResourceManager(typeof(FrmMain));
         	this.btnStart = new Button();
         	this.lblGu = new Label();
         	this.lblNu = new Label();
         	this.lblSu = new Label();
         	this.lblTu = new Label();
         	this.lblKu = new Label();
         	this.ppppp = new Label();
         	this.lblMu = new Label();
         	this.lblXu = new Label();
         	this.lblZu = new Label();
         	this.lblQu = new Label();
         	base.SuspendLayout();
         	this.btnStart.Location = new Point(165, 62);
         	this.btnStart.Name = "btnStart";
         	this.btnStart.Size = new Size(0, 0);
         	this.btnStart.TabIndex = 0; //
         	this.btnStart.UseVisualStyleBackColor = true;
         	this.btnStart.Click += this.btnStart_Click; //按钮事件
         	this.lblGu.Location = new Point(43, 123);
         	this.lblGu.Name = "lblGu";
         	this.lblGu.Size = new Size(53, 23);
         	this.lblGu.TabIndex = 1;
         	this.lblGu.Text = "label1";
         	this.lblNu.Location = new Point(90, 123);
         	this.lblNu.Name = "lblNu";
         	this.lblNu.Size = new Size(53, 23);
         	this.lblNu.TabIndex = 2;
         	this.lblNu.Text = "label2";
         	this.lblSu.Location = new Point(135, 123);
         	this.lblSu.Name = "lblSu";
         	this.lblSu.Size = new Size(53, 23);
         	this.lblSu.TabIndex = 3;
         	this.lblSu.Text = "label3";
         	this.lblTu.Location = new Point(182, 123);
         	this.lblTu.Name = "lblTu";
         	this.lblTu.Size = new Size(53, 23);
         	this.lblTu.TabIndex = 4;
         	this.lblTu.Text = "label4";
         	this.lblKu.Location = new Point(228, 123);
         	this.lblKu.Name = "lblKu";
         	this.lblKu.Size = new Size(53, 23);
         	this.lblKu.TabIndex = 5;
         	this.lblKu.Text = "label4";
         	this.ppppp.Location = new Point(278, 123);
         	this.ppppp.Name = "ppppp";
         	this.ppppp.Size = new Size(53, 23);
         	this.ppppp.TabIndex = 6;
         	this.ppppp.Text = "label4";
         	this.lblMu.Location = new Point(324, 123);
         	this.lblMu.Name = "lblMu";
         	this.lblMu.Size = new Size(53, 23);
         	this.lblMu.TabIndex = 7;
         	this.lblMu.Text = "label4";
         	this.lblXu.Location = new Point(369, 123);
         	this.lblXu.Name = "lblXu";
         	this.lblXu.Size = new Size(53, 23);
         	this.lblXu.TabIndex = 8;
         	this.lblXu.Text = "label4";
         	this.lblZu.Location = new Point(413, 123);
         	this.lblZu.Name = "lblZu";
         	this.lblZu.Size = new Size(53, 23);
         	this.lblZu.TabIndex = 9;
         	this.lblZu.Text = "label4";
         	this.lblQu.Location = new Point(457, 123);
         	this.lblQu.Name = "lblQu";
         	this.lblQu.Size = new Size(53, 23);
         	this.lblQu.TabIndex = 10;
         	this.lblQu.Text = "label4";
         	base.AutoScaleDimensions = new SizeF(7f, 12f);
         	base.AutoScaleMode = AutoScaleMode.Font;
         	base.ClientSize = new Size(626, 316);
         	base.Controls.Add(this.lblQu);
         	base.Controls.Add(this.lblZu);
         	base.Controls.Add(this.lblXu);
         	base.Controls.Add(this.lblMu);
         	base.Controls.Add(this.ppppp);
         	base.Controls.Add(this.lblKu);
         	base.Controls.Add(this.lblTu);
         	base.Controls.Add(this.lblSu);
         	base.Controls.Add(this.lblNu);
         	base.Controls.Add(this.lblGu);
         	base.Controls.Add(this.btnStart);
         	base.FormBorderStyle = FormBorderStyle.FixedSingle;
         	base.Icon = (Icon)componentResourceManager.GetObject("$this.Icon");
         	base.MaximizeBox = false;
         	base.Name = "FrmMain";
         	base.StartPosition = FormStartPosition.CenterScreen;
         	this.Text = "CSHOP";
         	base.Load += this.Form1_Load;
         	base.ResumeLayout(false);
         }
         ```

      2. Form1_Load：控件文字初始化

      3. btnStart_Click：当点击按钮时对Label的文字进行修改

         这串字符疑似Flag，而且也没有别的字符了，所以将其输入Auth，居然是Wrong。看来这几个字符有顺序问题。

         ```c#
         private void btnStart_Click(object sender, EventArgs e)
         {
         	this.lblSu.Text = "W";
         	this.lblGu.Text = "5";
         	this.lblNu.Text = "4";
         	this.lblKu.Text = "R";
         	this.lblZu.Text = "E";
         	this.lblMu.Text = "6";
         	this.lblTu.Text = "M";
         	this.ppppp.Text = "I";
         	this.lblGu.Text = "P";
         	this.lblQu.Text = "S";
         	this.ppppp.Text = "P";
         	this.lblTu.Text = "6";
         	this.lblXu.Text = "S";
         }
         ```

      4. 查看InitializeComponent中各标签的位置，发现都在同一行，从左到右分别是lblGu、lblNu、lblSu、lblTu、lblKu、ppppp、lblMu、lblXu、lblZu、lblQu，所以btnStart_Click中字符的顺序从左到右为P4W6RP6SES。需要注意的是，在btnStart_Click中出现的一个label有多次赋值的，要以最后一次赋值为准。

         PS.还有其他方法获得flag：

         - 因为按钮可以通过Tab来设置焦点，所以可以按Tab，然后按enter，就算是点击了按钮，即触发btnStart_Click，显示出flag

         - 使用spylite将按钮放到到铺满整个exe窗口，点击之后再缩小，flag就可以显示出来了。

           1. 打开cshop.exe，使用spylite获取其窗口。

              ![图1 spylite获取cshop.exe窗口](https://chrishuppor.github.io/image/Snipaste_2019-05-23_08-44-23.PNG)

           2. 在“窗口”中查看“子窗口”，很容易就能找到Button对应的子窗口。

              ![图2 子窗口列表](https://chrishuppor.github.io/image/Snipaste_2019-05-23_08-46-21.PNG)

           3. 选择button子窗口，进入“消息”选项，在窗口状态中选择”最大化“，此时会将button铺满整个客户区域。

              ![图3 窗口状态](https://chrishuppor.github.io/image/Snipaste_2019-05-23_08-47-36.PNG)

           4. 在cshop.exe区域中点击，然后将最大化去掉就可以看到flag了。

   ## 14.2 小结

   - 一个很简单的c#逆向题，dnSpy可以将其源码清晰的展现出来
     - dnSpy反编译的强大使我误以为c#程序的逻辑无处可藏，但通过咨询大佬得知没这么简单——c#程序可以附带自解压功能，将核心逻辑隐藏在资源节，在程序运行时自行解压该段代码并运行。然而dnSpy无法识别资源节的代码，此时需要用WinDbg进行动态调试。
     - OD无法调试c#程序，因为c#属于托管代码，由公共语言运行库环境运行，OD只能调试c等由os直接运行的非托管代码
   - 使用了“白纸白字”的方法隐藏了所有的label，使用0size的方法隐藏了按钮。
   - 其实理论上这个题也可以通过修改代码中btn的属性来使btn可见，从而点击按钮，但是我并没有实践成功。

# 15 PEPassword

   一个被输入框欺骗的题，一个很容易不知所云的题。

## 15.1 解题过程

1. 解压后没有看到readme，只有两个exe，一个叫original，一个叫packed。

	1. 运行original程序，弹出一个提示框，其中有”Password is“的字符，但是后面却是不可见字符。
	![图1 original程序界面](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-06-36.PNG)

	2. 运行packed程序，弹出一个输入框，提示输入password，可以随意输入字符，但却找不到check按钮。
	![图2 packed程序界面](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-07-39.PNG)

	3. 分析二者关系，从名称上看packed程序似乎是original程序加壳得到的。结合二者运行情况，可以猜测作者目的是用户在packed程序中输入一个正确的字符串，然后packed程序自解压成类似original的程序，弹出一个带有正确password的提示框。

	4. 拖入PEiD查看，果然original没壳，packed添加了PE password的壳。

2. 既然check在packed程序中，那么packed程序就是分析的重点。但在此之前我还是用OD查看了一下original程序。这个是一个十分简单的程序，除了提示字符串，程序中硬编码了一些字符，这些字符经过一轮异或操作后被MessageBox显示出来，只不过经过操作后这些字符都变成了不可见字符。因此，可以想到packed程序正确解密后会将这些字符修改成一些指定字符，经过变换后变为flag。

3. packed程序是加壳的程序，难以使用IDA分析，直接拖入OD。

	1. 查看packed程序使用的文本框字符获取函数
	之前使用PEView查看packed的IAT，只发现了LoadLibrary和GetProcAddress，说明这个壳会“手工”获取API地址。因此在GetProcAddress函数下断点就可以知道这个程序用到了哪些函数。
	
	结果没有发现文本框字符串获取函数。与字符获取有关的函数仅有GetMessageA。

	2. 看来Packed是使用GetMessageA逐个字符进行check。因此需要对GetMessageA下断点。注意，这里不可以用普通断点，因为GetMessageA是接收所有消息的，包括鼠标移动、定时器等，因此必须使用条件断点，在其获得键盘消息时中断。中断条件是[[ESP + 4] + 4] == 0x101.

	3. 程序会在用户输入字符时中断。此时查看堆栈可以找到返回地址0x409092.
	![图3 找返回地址](https://chrishuppor.github.io/image/Snipaste_2019-05-23_22-28-03.PNG)

	4. 运行至0x409092，单步运行程序，会发现之后调用了TranslateMessage和DispatchMessage。在0x4090AB处有一个比较，如果没有通过则会跳转回GetMessageA之前，所以这里应该是需要通过的。

		1. 既然是比较，如果不是硬编码的就一定有修改被比较值的地方，因此将程序拖入IDA，查看[EBP+0x402A3E]的引用。如下，发现仅在0x4091AB的位置对[EBP+0x402A3E]进行了inc操作。
		![图4](https://chrishuppor.github.io/image/Snipaste_2019-05-23_22-35-48.PNG)

		2. 在0x4091AB下断点，重新运行程序，输入字符，程序没有断下来。查看0x4091AB之前的代码，发现了一个jnz，所以可能是匹配失败所以跳走了。

		3. 在0x4091A6下断点，重新运行程序，输入字符，程序中断在0x4091A6。查看0x4091A6之前的代码发现了一个cmp。这个cmp之前有一个函数0x4091D8，猜测cmp中的EAX来自这个函数0x4091D8。

		4. 在IDA中查看sub_4091D8。模拟运行了一下，发现是一个hash函数，比较难求逆，所以这个函数应该不是突破口。从而这个路线应该都不是突破口。

	5. 爆破了0x4090AB之后，会发现之后有一个0x409200函数。跟进这个函数，如下，发现这个函数对0x401000起始的长度为0xfff字节的数据进行了修改。

         ![图5 sub_0x409200](https://chrishuppor.github.io/image/Snipaste_2019-05-23_22-58-47.PNG)
        
         其中，EBX是第一次经sub_4091DA计算得到的数据，EAX是第二次经sub_4091DA计算得到的数据，两次计算都与输入有关。因此猜测这里就是变相的check——如果当输入正确时，就可以将0x401000的代码解密正确，程序就可以弹出有正确pw的提示框；如果输入错误，程序就会跑飞。
        
         但是之前说过sub_4091DA函数为hash函数，难以求逆，看来需要其他手段使程序解密正确。
        
         进一步分析解密循环的寄存器的值发现，[EDI]的数据是一定的，ECX与EDX初始值确定为0，只有EAX和EBX的值是随输入变化的，因此如果能够知道EAX和EBX的初始值就能正确解密了。
        
         显然EAX是好求的。因为[EDI]解密结果由[EDI]原数据与EAX异或而来，已知解密结果应与original程序0x401000处代码一致，因此将packed程序0x401000起始的数据与original程序0x401000起始的数据异或就可以得到每轮循环中EAX的值。(注意内存数据存储使用小端序)
        
         EBX可以由EAX求得。解密循环开始前，EBX的初始值由硬编码的整数和输入经sub_4091DA函数计算而来，其中输入是未知的，所以这条路走不通。但是在解密循环中，如下，EAX第二个值由EAX初始值和EBX初始值计算而得，而且已知EAX的各个值，所以可以从这条路找到EBX初始值。
        
         ![图6 解密循环](https://chrishuppor.github.io/image/Snipaste_2019-05-24_16-46-54.PNG)
        
         将上图红框中的汇编代码整理的EAX2 = ROR4(EAX1 ^ROL4(EBX,  BYTE0(EAX1)), BYTE1(ROL4(EBX,  BYTE0(EAX1))))，其中EAX1表示计算前EAX值，EAX2表示计算后EAX值，ROR4(a,b)表示将32位的a循环右移b位，BYTE1(a)表示从低位开始取a的第二个字节，BYTE0(a)表示从低位开始取a的第一个字节。
        
         根据之前分析可知EAX1 =  0x014cec81 ^ 0xB6E62E17， EAX2 = 0x57560000 ^ 0x0d0c7e05，带入计算公式可得0x5a5a7e05 = ROR4(0xb7aac296 ^ROL4(EBX,  0x96), BYTE1(ROL4(EBX,  0x96)))。

	6. 求解ebx

		1. 令tmp = ROL4(EBX,  0x96)，则原公式简化为0x5a5a7e05 = ROR4(0xb7aac296 ^tmp, BYTE1(tmp))

		2. 找不到求解的办法，看来只能爆破了。如果从tmp入手，有0x100000000中可能，如果从BYTE1(tmp)入手有0x100中可能，因此从BYTE1(tmp)入手爆破。如果求得的tmp满足BYTE1(tmp)与假设的BYTE1(tmp)一致，则说明该tmp是正确的。

		3. 将0xb7aac296 ^tmp看做一个整体，令mid1 = 0xb7aac296 ^tmp，mid2 = BYTE1(tmp)则

            ```c
            EBX = ROR4(tmp, 0x96)
            tmp = 0xb7aac296 ^ mid
            mid1 = ROL4(0x5a5a7e05, mid2)
            ```

		4. 爆破脚本

            ```python
            a11 = 0x014cec81
            a21 = 0xB6E62E17
            
            a12 = 0x57560000
            a22 = 0x0d0c7e05
            
            #初始eax和第二个eax
            a1 = (a11 ^ a21)
            a2 = (a12 ^ a22)
            
            for mid2 in range(0, 0x100):
                count = mid2 % 32
                mid1 = (((a2 << count) | a2 >> (32 - count))) & 0xffffffff #注意这里，需要自己进行位数限制
                tmp = a1 ^ mid1
                if mid2 == ((tmp &0xff00)>>8):
                    count = 0x96 % 32
                    ebx = ((tmp >> count) | (tmp <<(32 - count))) & 0xffffffff
                    print (hex(ebx))
            ```

		5. 一共算出两个ebx，分别为0xa1beee22和0xc263a2cb。经测试，第二个ebx是正确的。

            ![图7 flagy](https://chrishuppor.github.io/image/Snipaste_2019-05-24_17-31-51.PNG)

		6. 最后对比了一下解密后的paecked和original的0x401000函数，如下，果然就是flag部分的字符不一样。

            ![图8 sub_401000对比](https://chrishuppor.github.io/image/Snipaste_2019-05-24_18-56-48.PNG)

## 15.2 小结

- 条件断点——条件表达式如何确定？

	- 断点条件一般是寄存器的值、函数参数的值等，在不确定的时候可以通过条件日志来确定。如图，选择从不中断程序、总是记录表达式的值。如果是函数入口，还可以选择总是记录函数参数。

       ![图9 条件日志配置](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-50-57.PNG)

	- 下完断点后，要清空原有日志。运行程序，查看日志中的数据，根据数据情况配置断点条件。例如本题中，在寻找GetMessageA参数中表示键盘消息的条件时，就用了这种方法。

		1. 总是记录函数参数，从不中断程序。运行程序后，分析其日志数据。

          ![图10 日志记录](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-57-18.PNG)

		2. 可以看到，函数第一个参数是一个指向MSG结构的指针，MSG结构中的Msg是WM_KEYUP时表示消息类型为按键。经查询，WM_KEYUP的值为0x101。也就是说中断条件是pMsg->Msg == 0x101。

		3. 从堆栈中可以看出，pMsg地址是ESP + 4，从MSG结构中可以看出Msg成员在MSG中偏移为0x4，所以pMsg->Msg可以表示为[[ESP + 4] + 4]。因此断点条件为[[ESP + 4] + 4] == 0x101。

	- 如果对断点条件表达式不是特别有把握，可以通过设置表达式来验证，比如将[[ESP + 4] + 4]设置为表达式，选择总是记录表达式的值。

- 本题关键有两点

	- 搞清楚破解的目的是让packed程序正确自解密成original的形式，但是字符串的值与original不同。
	- 突破定向思维，不与输入字符串纠缠，直接修改解密循环相关寄存器的初始值。

- 本题难点在于ebx爆破的方法

	- 查看他人的writeup时，发现除了爆破也没有其他的方法了。一开始我也直接从ebx开始爆破，但0x100000000的遍历似乎不太能接受，于是想到了从BYTE1(tmp)入手，计算复杂度陡降。

# 16 HateIntel

是一个mac系统的程序，之前从未见过mac相关的逆向，可能是我经验不够，不过这个题目难度与EasyELF有一拼。

## 16.1 解题步骤

1. 没有mac系统的机器，暂时不能动态调，先拖进IDA看看情况。

  1. 看上去很简单，函数很少，也能反编译。

  2. 查看字符串，有很明显的提示信息"Wrong"和"Correct"，查找其引用，发现了sub_2224函数。

  3. sub_2224
    代码如下，先接受了一个字符串输入，然后经过 calc_input_232C函数处理，再进入一个比较循环。如果比较通过就提示"Correct"，有一个失败就会提示"wrong"。这个循环显然是对input数据的check，因为&vars0 - 0x5C = &input_str，而且循环变量v3 = strlen(&input_str)，说不是对input数据的check都没有人信。查看byte_3004，发现都不是可见字符，说明之前对input进行了变换。恰巧在check之前有一个以input为参数的calc_input_232C函数。

         ```c
         int input_check_2224()
         {
           char input_str; // [sp+4h] [bp-5Ch]
           int v2_4; // [sp+54h] [bp-Ch]
           int v3; // [sp+58h] [bp-8h]
           int i; // [sp+5Ch] [bp-4h]
           char vars0; // [sp+60h] [bp+0h]
         
           v2_4 = 4;
           printf("Input key : ");
           scanf("%s", &input_str);
           v3 = strlen(&input_str);
           calc_input_232C((signed __int32)&input_str, v2_4);
           for ( i = 0; i < v3; ++i )
           {
             if ( *(&vars0 + i - 0x5C) != byte_3004[i] )
             {
               puts("Wrong Key! ");
               return 0;
             }
           }
           puts("Correct Key! ");
           return 0;
         }
         ```

    4. 查看calc_input_232C函数.

       核心逻辑如下，其中v2 = 4， v3 = &input。整个逻辑比较简单，只是内层循环跟正常人的写法不一样，翻译过来就是依次调用rol_2494函数处理输入的每个字符，整个字符串一共处理4次。

       那么处理的关键就是rol_2494函数了。

       ```c
       for ( i = 0; i < v2; ++i )
       {
           for ( j = 0; ; ++j )
           {
               input_str = strlen(v3);
               if ( input_str <= j )
                   break;
               v3[j] = rol_2494(v3[j], 1);
           }
       }
       ```

    5. 查看rol_2494函数

       又是一个具有迷惑性的一个函数，翻译过来就是将a1循环左移1位

       ```c
       int __fastcall ror_2494(unsigned __int8 a1, int a2_1)
       {
         int v3; // [sp+8h] [bp-8h]
         int i; // [sp+Ch] [bp-4h]
       
         v3 = a1;
         for ( i = 0; i < a2_1; ++i )
         {
           v3 *= 2; //左移一位
           if ( v3 & 0x100 ) //判断最高位是否为1
             v3 |= 1u; //如果最高位为1，则将末位置为1
         }
         return (unsigned __int8)v3;//返回8位的v3
       }
       ```

    6. 综上所述，calc_input_232C函数的作用就是依次将input中的字符循环左移4位，移位结果与byte_3004中数据比较。因此，flag就是依次将byte_3004中的数据循环右移4位。脚本如下：

       - byte_3004数据获取脚本

         ```python
         import idaapi
         ea = 0x3004
         
         res = []
         for i in range(0, 29):
             res.append((Byte(ea)))
             ea += 1
         
         print res
         ```

       - flag"翻译"脚本

         ```python
         a = [68, 246, 245, 87, 245, 198, 150, 182, 86, 245, 20, 37, 212, 245, 150, 230, 55, 71, 39, 87, 54, 71, 150, 3, 230, 243, 163, 146, 0]
         
         res = ''
         for item in a:
             tmp = (item >> 4) | ((item << 4)&0xff)
             res += chr(tmp)
         
         print (res)
         ```

## 16.2 小结

没什么好说的，只是想起了领袖的名言“一切反动派都是纸老虎”。无论题目怎么变化，本质是一样的。

- 要学会将不太正常的代码翻译成正常代码，这样就会简单许多。

# 17 x64Lotto

一个简单的多种解法的x64的题目。

## 17.1 解题过程

1. 使用ExeinfoPe查壳，没有问题。

2. 运行程序，要输入一些东西，估计是输入正确就返回Flag

3. 拖入IDA，先进入wmain函数，很容易发现整个程序的核心逻辑都在wmain函数中

	1. 首先，如下，接收6个输出，然后使用rand()生成6个随机数，将输入的6个数与随机生成的6个数进行比对。如果比对不通过则byte_1400035F0被赋0，否则byte_1400035F0为1。

         ```c
         v0 = time64(0i64);
         srand(v0);
         do
         {
             wprintf(L"\n\t\tL O T T O\t\t\n\n");
             wprintf(L"Input the number: ");
             wscanf_s(L"%d %d %d %d %d %d", &v13, &v14, &v15, &v16, &v17, &v18);
             wsystem(L"cls");
             Sleep(0x1F4u);
             v1 = 0i64;
             do
                 *(&v18 + ++v1) = rand() % 100;
             while ( v1 < 6 );
             v2 = 1;
             v3 = 0;
             v4 = 0i64;
             byte_1400035F0 = 1;
             while ( *(int *)((char *)&v19 + v4) == *(int *)((char *)&v13 + v4) )
             {
                 v4 += 4i64;
                 ++v3;
                 if ( v4 >= 24 )
                     goto LABEL_9;
             }
             v2 = 0;
             byte_1400035F0 = 0;
             LABEL_9:
             ;
         }
         while ( v3 != 6 );
         ```

	2. 接下来会看到一串硬编码的数据赋值，然后对这串数据进行处理，如下。其中，byte_140003021也是一串硬编码的数据。

         ```c
         do
         {
             v7 = byte_140003021[v6 - 1];
             v6 += 5i64;
             *((_WORD *)&v22 + v6 + 1) ^= (unsigned __int8)(v7 - 12);
             *((_WORD *)&v23 + v6) ^= (unsigned __int8)(byte_140003021[v6 - 5] - 12);
             *((_WORD *)&v23 + v6 + 1) ^= (unsigned __int8)(byte_140003021[v6 - 4] - 12);
             *((_WORD *)&v24 + v6) ^= (unsigned __int8)(byte_140003021[v6 - 3] - 12);
             *((_WORD *)&v24 + v6 + 1) ^= (unsigned __int8)(byte_140003021[v6 - 2] - 12);
         }
         while ( v6 < 25 );
         ```

	3. 接下来检查byte_1400035F0，如果byte_1400035F0为1则再次对v25起始的数据进行处理，并输出。否则就会输出v5起始的数据。显然，经过各种处理后的v25的数据就是flag，v5可能是错误提示。

         ```c
         if ( v2 )
         {
             v8 = 0;
             v9 = &v25;
             do
             {
                 v10 = *v9;
                 ++v9;
                 v11 = v8++ + (v10 ^ 0xF);
                 *(v9 - 1) = v11;
             }
             while ( v8 < 25 );
             v50 = 0;
             wprintf(L"%s\n", &v25);
         }
         ```

	4. 从wmain中可以看出，输入数据的作用就是通过检查，使byte_1400035F0值为1，不参与flag的生成。因此使用x64dbg直接爆破就可以了。（x64dbg是网友开发的OD的64位版）

         需要爆破的地方一共有两个，从反编译源码看一个是while ( v3 != 6 );一个是if ( v2 )，即RVA 0x1142和RVA 0x12E8。需要注意的是，程序在print flag之后就会退出，因此需要在退出代码前添加断点。爆破后程序运行结果如下。
    
         ![图1 Flag](https://chrishuppor.github.io/image/Snipaste_2019-05-29_11-55-34.PNG)

## 17.2 小结

- 这个题有多种解法，每种解法难度不同，其中自己计算v25的数据难度最大（也最蠢），主要考验逆向的灵活度。

- 查看他人的writeup，学到了一个更好的解法。

	- 原理

		当srand()参数相同时，rand()函数所用种子相同，进而生成的伪随机数序列相同。

       在同一秒内，调用time64(0)得到的值是一样的，因此只要输入时使用的伪随机数与程序自身生成随机数生成时间间隔小于1秒就能通过校验。

	- 做法

       利用echo输入参数并使用system执行cmd命令，代码如下

       ```c
       char com[1024] = { 0 };
       unsigned int t = _time64(0);
       int r[6] = { 0 };
       srand(t);
       for (int i = 0; i < 6; i++) {
           r[i] = rand() % 100;
       }
       sprintf_s(com, "echo %d %d %d %d %d %d | C:\\Users\\chris\\Downloads\\Lotto\\Lotto.exe", r[0], r[1], r[2], r[3], r[4], r[5]);
       system(com);
       ```

# 18 AutoHotKey2

还记得AutoHotKey1中投过的机吗，当年绕过的坎总要再走一遍。

## 18.1 解题过程

1. 查看readme得知本题的目标就是正确解密文件。打开程序发现没有输入的地方，说明需要对程序某些地方进行修改。

2. 使用UPX解压程序，将解压后的程序拖入IDA。

   1. 查找提示信息“EXE corrupted”，转到其引用。

      ![图1 EXE corrupted引用](https://chrishuppor.github.io/image/Snipaste_2019-05-29_18-53-38.PNG)

   2. 根据之前对AutoHotKey的分析可知，之所以会提示EXE corrupted，是因为解密文件时未通过校验，而校验函数正是sub_4508C7。只要sub_4508C7执行成功，程序就能正常运行了。

3. sub_4508C7函数分析

   1. 该函数首先调用sub_450F56函数初始化了一个数组，然后打开自身的磁盘文件，之后使用硬编码数据初始化了两个数组。

   2. 之后函数根据第三个函数的长度进行了一个循环操作。IDA中没有第三个参数的值，但通过OD发现，在进入循环前，这个参数为0，所以这个循环其实没有意义。

      ```C
      ffp[0x44] = 0;                                // 所以这一段根本没用
      for ( i = 0; i < hwnd_0; ++i )
          ffp[68] = (FILE *)((char *)ffp[68] + hwnd[i]);
      ```

   3. 然后从文件末尾读数据。

      1. 首先从倒数第8个字节开始读，读取一个大小为4字节的数据到ffp+1中。

         ```c
         fseek(*ffp, -8, SEEK_END);
         fread(ffp + 1, 4u, 1u, *ffp);                 // read 1 dword to ffp->cnt
         ```

         这里需要知道ffp变量的具体结构，也就是FILE的结构。在网上一直没有搜到FILE的定义，于是自己去FILE的头文件stdio.h中抠了，如下ffp + 1是ffp->_cnt的地址。也就是说从倒数第8个字节读取了一个DWORD大小数据存储到ffp->cnt中。

         ```c
         typedef struct _iobuf
         {
             char*   _ptr;
             int _cnt;
             char*   _base;
             int _flag;
             int _file;
             int _charbuf;
             int _bufsiz;
             char*   _tmpfname;
         } FILE;
         ```

      2. 然后继续读了4个字节的数据到v23中。

         ```c
         pos = ftell(*ffp);
         fp_tmp = *ffp;
         org_pos = pos;
         fread(&v23, 4u, 1u, fp_tmp);         // read 1 dword to v23
         ```

   4. 之后将指针设定到文件开头，逐个字符读取，并调用sub_450F95函数进行计算。其中，sub_450F95只是一个用来计算的函数，没有其他功能。

      ```c
      fseek(*ffp, 0, SEEK_SET);
      in_arga = 0;
      v11 = 0;
      if ( org_pos > 0 )
      {
          do
          {
              if ( (*ffp)->_flag & 0x10 )
                  break;
              fread(&v22, 1u, 1u, *ffp);
              v11 = calc_v22_450F95(&v17, v22);
              ++in_arga;
          }
          while ( (signed int)in_arga < org_pos );
      }
      if ( v23 != (v11 ^ 0xAAAAAAAA) )
          goto LABEL_25;
      ```

      显然这个循环是不需要破解的，因为这个文件有202 K个字符，如果这个循环有错误的话，那么这程序是无法破解的。因此，经过这个循环得到的v11就应该是正确的结果——如果if条件没通过则说明v23可能是错误的，需要根据v11来修改v23的值。

   5. 之后移动指针，从文件偏移ffp[1]处读取16个字节数据到v19中，然后将v19中的数据与之前初始化的两个数组数据进行比对。其中，ffp[1]就是之前的ffp->_cnt，也就是从文件尾部读出来的数据。

      ```c
      fseek(*ffp, (int)ffp[1], SEEK_SET);
      if ( !fread(v19, 0x10u, 1u, *ffp) )           // read 0x10B to v19
          goto LABEL_25;
      v12 = 0;
      do
      {
          if ( v19[v12] != tmp1_8[v12] ) //根据变量在堆栈的位置关系，这里会读到tmp2_8的数据
              break;
          ++v12;
      }
      while ( v12 < 16 );
      if ( v12 != 16 )//比对失败
      {
      LABEL_25:
          v16 = 3;
      LABEL_19:
          v13 = v16;
          fclose(*ffp);
          return v13;
      }
      ```

      在动态调试时，fseek会失败，因为ffp[1]的值不正确。那么如何得到正确的ffp[1]呢？既然要求v19的数据与tmp1_8数据相同，那么在文件中搜索tmp1_8数据出现的起始位置不就是ffp[1]吗？在文件中搜索tmp1_8，发现其起始位置偏移为0x32800，说明文件倒数第8-5个字节为00 28 03 00（注意使用小端序）。

   6. 接下来继续读一个字符且要求这个字符为0x3。查看文件中数据0x32800+0x10+1处的数据的确为0x3，不需要修改。

      ```c
      fread(&v26, 1u, 1u, *ffp);
      if ( v26 != 3 )
      {
          v16 = 4;
          goto LABEL_19;
      }
      ```

      1. 所有检查都通过后再进行一波计算，然后返回0。先假定这波计算没有问题。

         ```c
         fread(&v24, 4u, 1u, *ffp);
         v14 = v24 ^ 0xFAC1;
         fread(ffp + 3, 1u, v24 ^ 0xFAC1, *ffp);
         sub_450ABA((int)(ffp + 3), v14, v14 + 0xC3D2);
         *((_BYTE *)ffp + v14 + 12) = 0;
         v15 = 0;
         for ( ffp[68] = 0; v15 < v14; ++v15 )
             ffp[68] = (FILE *)((char *)ffp[68] + *((char *)ffp + v15 + 12));
         ffp[2] = (FILE *)ftell(*ffp);
         return 0；
         ```

   7. 至此已经很清楚要做什么了——修改最后8个字节的数据。这8字节数据被分为两个DWORD变量来处理。

      - 第一个DWORD——指出v19的偏移，已知应为0x32800，因此文件中数据为00 28 03 00
      - 第二个DWORD——通过v23的校验，应为v11^0xAAAAAAAA，可以通过OD从内存中获取相应数据。

      需要注意的是，v11的值与除最后四字节数据的全部文件数据相关，所以要先修改第一个DWORD数据，然后再从OD中获取v23的正确值。

4. 将未解压的程序拖进OD，UPX的壳可以使用OD的自解压功能自动脱壳。

   通过IDA可以知道v23的校验代码地址为0x4509DD。OD自解压完成后在该处下断点直接运行，可以得到v23应为0x362BC633。因此，最后四个字节为33 C6 2B 36。

5. 将文件修改好，直接运行得到如下信息。Flag就是这段话描述的人的名字，而且是小写不带空格的。*（权游爱好者实锤了）*

   ![图2 Flag信息](https://chrishuppor.github.io/image/Snipaste_2019-05-30_09-30-11.PNG)

## 18.2 启示

这个题没有过多的启示，主要是动静结合的分析以及见招拆招，考验做题人头脑的灵活性。

- 在reverse中，如果某部分算法比较复杂，尤其是难以求逆的那种，无论是逻辑还是内部数值，大概率是没有问题的。一般需要破解的是简单算法的求逆和复杂逻辑算法的参数。

# 19 FlashEncrpyt

一个地地道道的Misc签到题。

## 19.1 解题过程

1. 将文件拖入flash分析工具FFDec.

   如下，看到了很多东西，尽管不懂flash，但凭经验也知道关键在于脚本。

   ![图1 加载swf](https://chrishuppor.github.io/image/Snipaste_2019-05-30_16-35-45.PNG)

2. 查看脚本，发现其中的代码很混乱，看来是加了混淆。FFDec也提示使用设置中的自动去混淆功能。

   ![图2 自动去混淆](https://chrishuppor.github.io/image/Snipaste_2019-05-30_16-40-57.PNG)

3. 去混淆后，脚本代码就变得十分清晰了。用户输入数据到文本框中，点击按钮触发脚本。如果用户输入的数据通过匹配，则调用函数转到对应的帧。如下，如果输入是1456，点击按钮后则转到frame3.

   ![图3 脚本代码示例](https://chrishuppor.github.io/image/Snipaste_2019-05-30_16-42-36.PNG)

4. 从脚本中可以得知每一帧对应的数字，从第一帧开始逐个输入，最终会转到frame7，并显示出Flag.

## 19.2 启示

- FFDec确实是个好使的flash分析软件
- 不管什么类型的文件，运行在什么平台，使用什么语言写的，逆向的本质就是找出功能与代码的对应关系。

# 20 CSHARP

是一个动态解密函数的程序。

## 20.1 解题过程

1. 拖入PEiD得知这是一个C#程序。

2. 拖入dnSpy，查看入口函数。

   这个函数十分简单，就是创建了一个Form1的对象。

   ```c#
   private static void Main()
   {
       Application.EnableVisualStyles();
       Application.SetCompatibleTextRenderingDefault(false);
       Application.Run(new Form1());
   }
   ```

3. 查看Form1构造函数

   如下，首先获取了MetMett函数的二进制代码并赋值给了Form1.bb，然后对Form1.bb进行了一些变换，可能是对MetMett代码的解密操作。

   ```c#
   public Form1()
   {
       Form1.bb = base.GetType().GetMethod("MetMett", BindingFlags.Static | BindingFlags.NonPublic).GetMethodBody().GetILAsByteArray();
       byte b = 0;
       for (int i = 0; i < Form1.bb.Length; i++)
       {
           byte[] array = Form1.bb;
           int num = i;
           array[num] += 1;
           b += Form1.bb[i];
       }
       Form1.bb[18] = b - 38;
       Form1.bb[35] = b - 3;
       Form1.bb[52] = (b ^ 39);
       Form1.bb[69] = b - 21;
       Form1.bb[87] = 71 - b;
       Form1.bb[124] = (b ^ 114);
       Form1.bb[141] = (b ^ 80);
       Form1.bb[159] = 235 - b;
       Form1.bb[179] = 106 + b;
       Form1.bb[200] = 36 - b;
       Form1.bb[220] = b - 3;
       this.InitializeComponent();
   }
   ```

4. 查看MetMett，果然这个函数代码有问题，所以 Form1()就是对这段代码进行了解密。但是 Form1()并没有将解密结果写回这个函数，需要关注一下Form1.bb的使用。

5. 查看btnCheck_Click函数，发现它只调用了MetMetMet函数，并且以用户输入为参数。说明MetMetMet函数就是匹配的核心函数。

   ```c#
   Form1.MetMetMet(this.txtAnswer.Text);
   ```

6. 查看MetMetMet函数

   1. 这个函数首先将用户输入转为base64编码字符串。

      ```c#
      byte[] bytes = Encoding.ASCII.GetBytes(Convert.ToBase64String(Encoding.ASCII.GetBytes(sss)));
      ```

   2. 然后使用TypeBuilder和MethodBuilder创建了一个名为MetM的动态方法，这个方法的代码就是Form1.bb。

      ```c#
      TypeBuilder typeBuilder = AppDomain.CurrentDomain.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.RunAndSave).DefineDynamicModule(assemblyName.Name, assemblyName.Name + ".exe").DefineType("RevKrT1", TypeAttributes.Public);
      MethodBuilder methodBuilder = typeBuilder.DefineMethod("MetMet", MethodAttributes.Private | MethodAttributes.Static, CallingConventions.Standard, null, null);
      TypeBuilder typeBuilder2 = AppDomain.CurrentDomain.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.RunAndSave).DefineDynamicModule(assemblyName.Name, assemblyName.Name + ".exe").DefineType("RevKrT2", TypeAttributes.Public);
      //here
      typeBuilder2.DefineMethod("MetM", MethodAttributes.Private | MethodAttributes.Static, CallingConventions.Standard, null, new Type[]
                                    {
                                        typeof(byte[]),
                                        typeof(byte[])
                                    }).CreateMethodBody(Form1.bb, Form1.bb.Length); 
      Type type = typeBuilder2.CreateType();
      MethodInfo method = type.GetMethod("MetM", BindingFlags.Static | BindingFlags.NonPublic);
      ```

   3. 然后以[1,2]和base64编码的输入为参数调用了MetM函数

      ```c#
      method.Invoke(obj, new object[]{array, bytes});
      ```

   4. 最后根据array[0]的值判断匹配是否通过。由此可见，匹配的核心代码就是Form1.bb中的数据。直接看二进制数据实在太难了，最好是有反编译源码——只要将Form1.bb中的数据替换掉文件中的MetMett函数的数据就可以在dnSpy看到反编译源码了。

7. 在Form1构造函数下断点，解密完成后，利用dnSpy从内存中获取Form1.bb的数据。这个数据就是MetMett函数正确的二进制数据。

8. 在exe文件中定位MetMett函数位置。这个位置不是dnSpy显示的0x52C，而需要根据MetMett函数二进制数据自己查找。然后将该函数代码替换为正确的代码。

   替换脚本：

   ```python
   f1 = open("CSharp.exe", 'rb')
   b1 = f1.read();
   f1.close()
   
   fw = open("changed.exe", "wb")
   
   fr = open("MetMett.txt", "r")
   text = fr.read()
   fr.close()
   
   a = []
   tmp = 0
   for i in range(0,len(text)):
       if i % 2 == 1:
           tmp = tmp * 16 + int(text[i], 16)
           a.append(tmp)
           else:
               tmp = int(text[i], 16)
   
               b = bytearray(b1)
               for i in range(0x538, 0x538 + 234):
                   b[i] = a[i - 0x538]
                   
   fw.write(b)
   fw.close()
   ```

9. dnSpy打开解密后的exe文件，MetMett函数反编译代码如下，其中bt为输入字符串的base64编码数据，chk是数组[1,2]。

   ```c#
   private static void MetMett(byte[] chk, byte[] bt)
   {
       if (bt.Length == 12)
       {
           chk[0] = 2;
           if ((bt[0] ^ 16) != 74)
           {
               chk[0] = 1;
           }
           if ((bt[3] ^ 51) != 70)
           {
               chk[0] = 1;
           }
           if ((bt[1] ^ 17) != 87)
           {
               chk[0] = 1;
           }
           if ((bt[2] ^ 33) != 77)
           {
               chk[0] = 1;
           }
           if ((bt[11] ^ 17) != 44)
           {
               chk[0] = 1;
           }
           if ((bt[8] ^ 144) != 241)
           {
               chk[0] = 1;
           }
           if ((bt[4] ^ 68) != 29)
           {
               chk[0] = 1;
           }
           if ((bt[5] ^ 102) != 49)
           {
               chk[0] = 1;
           }
           if ((bt[9] ^ 181) != 226)
           {
               chk[0] = 1;
           }
           if ((bt[7] ^ 160) != 238)
           {
               chk[0] = 1;
           }
           if ((bt[10] ^ 238) != 163)
           {
               chk[0] = 1;
           }
           if ((bt[6] ^ 51) != 117)
           {
               chk[0] = 1;
           }
       }
   }
   ```

   匹配逻辑十分简单，解密脚本如下：

   ```python
   b = [74, 70, 87, 77, 44, 241, 29, 49, 226, 238, 163, 117]
   x = [16, 51, 17, 33, 17, 144, 68, 102, 181, 160, 238, 51]
   pos = [0, 3, 1, 2, 11, 8, 4, 5, 9, 7, 10, 6]
   
   res = [0 for i in range(12)]
   for i in range(12):
       res[pos[i]] = b[i] ^ x[i]
   
   resstr = ''
   for i in range(12):
       resstr += chr(res[i])
   print (resstr)
   ```

## 20.2 小结

- MetMett函数二进制代码解密算法十分简单，因此也可以自己脚本解密。
- MetMett函数在exe文件中的地址要自己查，dnSpy给出的不靠谱。

# 21 MetroApp

一个我觉得学了也没有用的题目。

## 21.1 解题过程

1. 安装MetroApp程序，查看程序运行情况。

   与其他验证码程序的使用一样——用户输入一串字符串，点击check按钮，程序判断用户输入是否正确。

2. 解压appx文件，得到一个exe程序。使用IDA查看这个exe程序，有很多函数，有没有明显的字符串特征。

3. 运行appx文件，使用52专用OD附加进程。

   1. 再次查看字符串，发现了“Wrong”“Correct”这种字符串，转到引用，发现其在函数sub_413BC0中。

4. 在IDA中查看sub_413BC0函数逻辑。发现其在提示“Wrong”和“Correct”之前会调用WindowsCompareStringOrdinal函数进行比对，比对的两个字符串一个是MERONG，另一个未知。使用OD追踪可知另一个参数是用户输入。如果用户输入为“MERONG”则会进入correct提示信息的构造，否则进入wrong信息的构造。

   所以MERONG是flag?提交后发现不是。仔细看的话会发现这里只是使用WindowsCreateStringReference创建了字符串，所以可能之后还有检查，要通过后才会弹出提示框。

5. 在IDA中查看“Wrong”和“Correct”字符串构造后的代码。

   1. 从OD中可以知道调用sub_40F4E0后，返回值变为指向用户输入的指针，所以sub_40F4E0大概就是用于获取用户输入的，然后会进入一个while(1)循环。

   2. 从OD中可以知道调用WindowsGetStringLen的参数为用户输入的指针，返回值为用户输入的长度，因此可以知道程序中有一个与用户输入长度进行比较的变量v80。

      查看这个变量的引用发现其初始值为0，每轮循环++，如果>=用户输入长度则跳出循环，可见是循环控制变量，循环的次数就是用户输入长度。

      ```c
      input_len = WindowsGetStringLen(tmp2_input);
      v37 = v80 < input_len;
      ```

   3. 从OD中可以知道WindowsGetStringRawBuffer时的参数均为用户输入的指针，因此如下的v50 v51 v52都是指向用户输入字符串的指针。

      ```c
      v50 = WindowsGetStringRawBuffer(input_object_tmp1, 0);
      v51 = WindowsGetStringRawBuffer(input_object_tmp2, 0);
      v52 = WindowsGetStringRawBuffer(input_object_tmp3, 0);
      ```

   4. 那么如下计算式就是对用户输入的操作。因为程序使用的是UNICODE，所以逐个处理字符时+2而不是+1。

      ```c
      LOBYTE(v1) = *(_WORD *)(v50 + 2 * v80 + 2) != (unsigned __int8)(byte_4307A8[v80 & 7] ^ __ROL1__(*(_BYTE *)(v52 + 2 * v80),*(_WORD *)(v51 + 2 * v80) & 7));
      ```

      翻译过来就是

      ```c
      if (input[v80 + 1] != byte_4307A8[v80 & 7]^rol1( *(BYTE*)input[v80], (input[v80] & 7) ))
      	LOBYTE(v1) = TRUE
      else
      	LOBYTE(v1) = FALSE
      ```

   5. 如果v1不为零，则进行一系列操作，否则直接v80++，进入下一轮循环。那么什么时候跳出循环呢？因为这里有一堆虚函数和一堆错误后的break，所以难以用IDA直接看出跳出循环的条件。用OD追踪，如果jmp的地址不为loc_414054则说明跳出循环了。

      事实证明，当且仅当v80 = input_len时会使用goto跳出本函数的循环。

   6. 猜测应该是要v1 = FALSE，因为如果v1是TRUE，则无法求逆获得正确的flag，即使爆破，flag的可能值会有很多。

      爆破v1,将0x4142F9处的je改为jmp后无论输入什么都会弹出correct。说明确实要求v1为FALSE，也就是input[v80 + 1] == byte_4307A8[v80 & 7]^rol1( *(BYTE*)input[v80], (input[v80] & 7) )

      ![图1 爆破v1](https://chrishuppor.github.io/image/Snipaste_2019-06-06_11-17-12.PNG)

   7. 看来只能爆破获得满足等式的字符串了。需要注意，readme中说flag是大写字母和数字的组合。

      爆破脚本如下。

      ```python
      def rol(i, n, m, bits):
          tmp = (i << n)
          return (tmp & m)|(tmp >> bits)
      
      static_a = [119, 173, 7, 2, 165, 0, 41, 153, 40, 41, 36, 94, 46, 42, 43, 63, 91, 93, 124, 92, 45, 123, 125, 44, 58, 61, 33, 10, 13, 8, 0, 0]
      
      table = "QWERTYUIOPASDFGHJKLZXCVBNM0123456789"
      
      for item in table:
          count = 0;
          tmp = item
          s = tmp
          while(1):
              tmp = static_a[count & 7]^rol(ord(s[count]), (ord(s[count])&7), 0xff, 8)
              if chr(tmp) not in table:
                  break
              s += chr(tmp)
              count += 1
          if len(s) > 1:
              print(s)
      ```

      输出结果如下，只有D34DF4C3是正确的，可能是有未发现的长度限制。

      ```python
      T2
      D34DF4C3
      0G
      44
      8O
      ```

## 21.2 参考文章

- MetroApp程序安装
  - [【UWP开发】uwp应用安装失败](https://blog.csdn.net/egostudio/article/details/77408126)
  - [如何在Windows 10中手动安装.appx文件](https://www.sysgeek.cn/windows-10-install-appx-uwp/)

## 21.3 小结

- string.exe和IDA string功能得到的字符串结果不同。
- 虽然最后找到了Flag，但仍然不知道为什么就能弹出correct。查看了别人的WP，大佬们也没有给出更多的信息，而且基本也是迷迷糊糊的。怎么说呢？感觉逆向除了技术，更多的是靠社工的推理（连蒙带猜）。

# 22 Multiplicative

一个依赖于工具的简单的java逆向题。

## 22.1 解题过程

1. 使用jd-gui查看jar时什么也没有，换jadx查看jar时可以看到main函数。代码如下：

   ```java
   public class JavaCrackMe {
       public static final synchronized /* bridge */ /* synthetic */ void main(String... strArr) {
           synchronized (JavaCrackMe.class) {
               try {
                   System.out.println("Reversing.Kr CrackMe!!");
                   System.out.println("-----------------------------");
                   System.out.println("The idea came out of the warsaw's crackme");
                   System.out.println("-----------------------------\n");
                   if (Long.decode(strArr[0]).longValue() * 26729 == -1536092243306511225L) {
                       System.out.println("Correct!");
                   } else {
                       System.out.println("Wrong");
                   }
               } catch (Exception e) {
                   System.out.println("Please enter a 64bit signed int");
               }
           }
           return;
       }
   }
   ```

   整个函数十分简单，只要满足 (Long.decode(strArr[0]).longValue() * 26729 == -1536092243306511225L) 就可以了。

2. 计算 -1536092243306511225/26729，结果居然是小数。可见Long.decode(strArr[0]).longValue() * 26729的结果是溢出了。已知低64位数据为 -1536092243306511225/26729，也就是( -1536092243306511225/26729 & 0xffffffffffffffff)，高位数据需要爆破。

   ```python
   y = -1536092243306511225
   y = (y & 0xffffffffffffffff) #将有符号的y转换为无符号整数，因为符号位溢出了，所以剩下的这部分都是数值。
   
   i = 1 #高位
   while(1):
       k = (i << 64 )| y
       if(k % 26729 == 0):  #这里也可以用 k//26729 * 26729 == k来控制，但肯定k % 26729 == 0运算更快
           tmp = k//26729
           if(tmp > 2 ** 63):# 因为有提示信息说输入的是64bit有符号数，所以要将tmp转为有符号数
               tmp = tmp - 2**64
           print('%d'% (tmp))
           break
       i += 1
   ```

   输出的tmp即flag。

## 22.2 工具下载

- [jadx下载地址](https://www.softpedia.com/get/Programming/Other-Programming-Files/Jadx.shtml)

## 22.3 小结

- python中的数学运算符号

  - python中" / "就表示 浮点数除法，返回浮点结果，" // "表示整数除法。本题应用'//'.
  - python中"*"就表示 乘法，“**”表示乘方。

- python中的有符号整数和无符号整数*（PS.已经被坑过两次了）*

  - 将有符号整数转换为无符号整数

    - 例如将32位的x转换为无符号数——(x & 0xffffffff)

  - 将无符号整数转换为有符号整数

    - 例如将32位的x转换为有符号数

      ```python
      if x > 2 ** 31:
          return (x-2**32)
      else:
          return x
      ```

- 工欲善其事必先利其器。同种工具有很多，这个不行换那个。

# 23 SimpleVM

一个加了VM壳的ELF程序，不是很simple，用到了pin和IDA远程调试。

## 23.1 解题过程

1. 运行程序，提示Access Denied。使用root账户运行程序，发现可以正常运行。所以这个程序要求root权限。

   运行后提示用户输入，如果输入错误则返回Wrong。

2. 程序直接拖进IDA，发现无法解析，果然是加了壳。根据题目名称可知加了VM壳。

3. 黔驴技穷了，只好去看大佬的WP，看懂了再复现一遍。

## 23.2 复现过程

1. 使用IDA远程调试程序，从内存中将脱壳后的代码数据dump出来。

   如图，经查看发现debug001是ELF文件的数据部分，debug002应该就是代码段。

   ![图1 远程调试内存分段情况](https://chrishuppor.github.io/image/Snipaste_2019-06-08_22-55-42.PNG)

   使用idapython脚本获得dump数据，代码如下。

   ```python
   import idaapi
   
   fw = open("SimpleVM_dump.hex", "wb")
   beginea = 0x8048000
   endea = 0x804c000
   for i in range(0, endea - beginea):
   	ea = beginea + i
   	bytess=struct.pack('B',Byte(ea))
   	fw.write(bytess)
   
   fw.close()
   ```

2. 将dump后的数据文件拖入IDA，查看strings，发现了Input。查找Input的引用则进入到sub_8048556函数。猜测这个函数就是主要函数了。

   1. 根据Input可以猜出sub_8048460是print函数，进而可以得到804B07A的数据用于生成Correct字符串，804B074的数据用于生成Wrong字符串，804B060的数据用于生成AeccessDenied字符串。

   2. 有很多函数最后都是调用一个dword_XXXX，静态分析无法知道这是个什么，但是从动态分析的内存中可以得到该地址存储的数据。这些地址一般存储的都是库函数的地址。库的函数名称及偏移列表可以通过nm命令在linux中获得，例如```nm -a /lib/i386-linux-gnu/ld-2.24.so > ~/Desktop/1.txt```。根据库函数的地址和库的基址计算出其偏移，然后去列表中比对就可以知道调用的那个函数了。

      比对结果：

      sub_80489FE->gguid，根据AeccessDenied可知该函数用于判断权限。

      sub_804B01C->pipe

      sub_804B020->fork

      sub_804B00->与IO_stdout相关。

   3. 至此就可以得到比较清晰的程序逻辑了。

      使用fork生成子进程，使用pipe创建管道用于子进程和父进程通信。父进程从stdin接受用户输入数据，并通过管道传递给子进程，同时将804B0A0数据传给子进程，由子进程判断输入是否正确。

      - 父进程接收用户输入时限制输入长度小于9.

        ```c
        input = 0;
        v11 = 0;
        v12 = 0;
        read_8048400(0, (int)&input, 10);       // read from stdin,maxbytes is 10
        if ( (_BYTE)v12 )//这里要求输入不能覆盖v12，也就是长度小于9.
        ```

      - 子进程调用sub_8048C6D判断输入是否正确。该函数虽然比较复杂，但是也是可以看的。（参考WP [Reversing.kr题目之SimpleVM详解](https://www.freebuf.com/news/164664.html)）。不过我选择了更省力的方法——pintools

3. 通过阅读invicsfate大佬的WP，清楚了本题边信道攻击的原理，编写了如下脚本进行爆破。

   需要注意的是，inscount1.cpp中结果是输出到文件中的，需要自行修改为输出到stdout；execlCommand中在调用fin.readline()获取返回数据时，三个返回信息的顺序是不确定的，也就说不能确保result1就是程序的打印信息，也不能确保result3就是pintools的返回数据。

   ```python
   #!/usr/bin/env python
   #-*- coding:utf-8 -*-
   
   import popen2,string
   
   INFILE = "fx"
   CMD = "./pin -t source/tools/ManualExamples/obj-ia32/inscount1.so -- ./SimpleVM <" + INFILE
   
   def execlCommand(command):
   	fin,fout = popen2.popen2(command)
   	result1 = fin.readline()
   	result2 = fin.readline()
   	result3 = fin.readline()
   	fin.close()
   	return result1,result2,result3
   
   choices = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#$%&'()*+,-./:;<=>?@[\]^_`{|}~"
   
   def writefile(data):
   	f = open(INFILE,'w')
   	f.write(data)
   	f.close()
   
   def findRealCount(res1, res2, res3):
   	mincount = 0;
   	if "count" in res1:
   		count = int(res1[len("count"):len(res1)-1])
   		if mincount == 0:
   			mincount = count
   
   	if "count" in res2:
   		count = int(res2[len("count"):len(res2)-1])
   		if mincount == 0:
   			mincount = count
   		elif count < mincount:
   			mincount = count
   
   	if "count" in res3:
   		count = int(res3[len("count"):len(res3)-1])
   		if mincount == 0:
   			mincount = count
   		elif count < mincount:
   			mincount = count
   
   	return mincount
   
   
   #-------------------------------------------------------
   curkey = ''
   while(1):
   	print "[+]cur key-\"" + curkey + "\""
   
   	maxCount = 0;
   	tmpkey = curkey
   	lastkey = curkey
   
   	for char in choices:
   		tmpkey = lastkey + char
   		writefile(tmpkey)
   		res1, res2, res3 = execlCommand(CMD)
   		print ">", tmpkey, '\n', res1, res2, res3, "maxcount:", maxCount, "\n"
   		if "Wrong" not in (res1 + res2 + res3):
   			print "[+]the key is : ", tmpkey
   			exit()
   
   		curcount = findRealCount(res1, res2, res3)
   		if curcount > maxCount:
   			maxCount = curcount
   			curkey = tmpkey
   ```

   运行示例：

   ![图2 运行示例](https://chrishuppor.github.io/image/Snipaste_2019-06-08_23-38-07.PNG)

## 23.3 参考文档

- [windows下使用IDA远程调试linux(ubuntu)下编译的程序](https://blog.csdn.net/lacoucou/article/details/71079552)

## 23.4 参考WP

- [Reversing.kr题目之SimpleVM详解](https://www.freebuf.com/news/164664.html)
- [171012 逆向-Reversing.kr（SimpleVM)](https://blog.csdn.net/whklhhhh/article/details/78221365)
- [invicsfate-Reversing.kr](http://invicsfate.cc/2017/09/18/reversing-kr/?nsukey=7EFkQUzSz2nFYRDmRCeolHMtQnmzZBoGYHUU50QI7Bc0xMkHYUIBvpOYNYXJDfrzX4pfToRQKC0iR9Vuzb0y42tJYAvOAmpKODlZ82UZKKYrY639VbLrXMa69bX2Ycyb%2FNlQkylQ23gn2rpzL2wTMyn0lEwegX2xrByI4fkkIwv9u3aZsbMgcRC2D9XNX1iPdo5DrhoVlI%2BFbh9S9xCs0A%3D%3D)

## 23.5 小结

- IDA远程调试需要在远程启动相应的server，这个server文件在IDA目录的dbgsrv文件夹中。
- linux下pin安装要对应好版本（gcc版本和系统位数），否则在编译时就是一堆堆的err。以下是我的一些经验：
  - 在ubuntu 18.04 64位机器上安装pin-3.7，可以编译目标平台为intel64的程序，但是ia32的程序就各种报错。
  - 在ubuntu 16.04 32位机器上安装pin-2.7，因为gcc版本不匹配报错；安装pin-3.7，十分舒适。

## 23.6 其他

[[脚本及IDB文件下载](https://github.com/chrishuppor/attachToBlog/tree/master/SimpleVM.idb)]

# 24 CRC1

难点在于能否想到中间相遇攻击。*（是剩下的四个题中最好捏的了，其余的捏的手都疼了）*

## 24.1 解题过程

1. 运行一下程序。

   需要用户输入数据，然后程序显示用户输入的数据WRONG还是CORRECT。

2. 将程序拖入IDA，在strings中找到WRONG，然后转到WRONG的调用。WRONG在函数sub_401070中被调用。

3. sub_401070函数较为简单，从头看起。

   1. 首先使用GetDlgItemTextA获取了一个字符串，要求这个字符串大小为8 Bytes。

      ```c
      GetDlgItemTextA(hDlg, 1000, &input0, 20) != 8
      ```

      使用RH查看就可以知道，ID为1000的控件就是那个文字输入框。

      ![图1 资源视图](https://chrishuppor.github.io/image/Snipaste_2019-06-11_14-19-51.PNG)

      由此可知，正确输入一共有8个字符。

   2. 接下来是一个略带蛇皮走位的赋值操作。

      ```c
      len_init = strlen(aHelloWelcomeTo) + 1;
      len = len_init;
      len_init >>= 2;
      qmemcpy(&szChar, aHelloWelcomeTo, 4 * len_init);
      ptr_aHelloWelcomeTo = &aHelloWelcomeTo[4 * len_init];
      ptr_szChar = &szChar + 4 * len_init66;
      v_2 = len & 3;
      v6 = 0;
      qmemcpy(ptr_szChar, ptr_aHelloWelcomeTo, v_2);
      v7 = &szChar;
      ```

      翻译过来就是strncpy(&szChar, aHelloWelcomeTo, strlen(aHelloWelcomeTo)); v7 = &szChar。

      简化一下就是strncpy(v7, aHelloWelcomeTo, strlen(aHelloWelcomeTo));

   3. 然后就是将用户输入引入v7数据。

      ```c
      do
      {
          v8 = *(&input0 + v6++);
          *v7 = v8;
          v7 += 16;
      }
      while ( v6 < 8 );
      ```

      这里将v7看做8*16的矩阵，将v7[i\][0]赋值为input[i]

   4. 然后对v7中数据进行了计算。

      ```c
      do
      {
          v11 = (unsigned __int8)v9 ^ (unsigned __int8)*(&szChar + v10);
          v12 = v9 >> 8;
          LODWORD(v9) = v12 ^ dword_4085E8[2 * v11];
          v13 = HIDWORD(v9) ^ dword_4085EC[2 * v11];
          ++v10;
          HIDWORD(v9) = v13;
      }
      while ( v10 < 256 );
      ```

      其中&szChar就是v7，并且根据v10 < 256可知v7的大小为256字节。根据后面的代码可知，最后得出的v9和v13就是匹配成功的关键。dword_4085EC[2 * v11]就是dword_4085E8[2 * v11 + 1]。

      查看dword_4085E8引用，发现在sub_401000函数中对其进行了赋值。很容易发现sub_401000函数其实就是crc_table的初始化函数，因此dword_4085E8数组就是crc_table的存储数组。

      另外，在sub_401000中还有对aHelloWelcome数据的修改。*（真是的，蛇皮走位的赋值还不行，还有犄角旮旯的数据修改，这就不怎么磊落了）*

   5. 接下来就是匹配的关键了：if ( (_DWORD)v9 != 0x5F695F6C || v13 != 0x676F5F67 )。当且仅当v9 == 0x5F695F6C  && v13 == 0x676F5F67 时程序才会提示出CORRECT。

      ```c
      do
      {
          v11 = (unsigned __int8)v9 ^ (unsigned __int8)*(&szChar + v10);
          LODWORD(v9) = (v9 >> 8) ^ dword_4085E8[2 * v11];
          HIDWORD(v9) = HIDWORD(v9) ^ dword_4085E8[2 * v11];
          ++v10;
      }
      while ( v10 < 256 );
      ```

4. 破解

   1. 使用IDA动态加载程序，使用IDAPython从内存中获取dword_4085E8数组和szChar原始数据。

      emmm...IDA崩了，所以自己手动算吧。

   2. 根据CRC原理设计中间相遇攻击，降低爆破空间，进而计算出flag。*(CRC原理简介可参考crc学习笔记)*

      脚本如下:

      ```python
      def initCRCtable(dividend):
      	crc_table = []
      	index_table = []
      	for i in range(0, 256):
      		index_table.append(0)
      
      	for i in range(0, 256):
      		crc = i
      		for j in range(0, 8):
      			tmp = crc & 0x1
      			crc >>= 1
      			if tmp == 1:
      				crc ^= dividend
      		crc_table.append(crc) #for improve crc_calc rapid
      		index_table[crc >> 56] = i #to find the value of crc.table[index] with the result of (lastcrc ^ crc.table[index]), so we can know lastcrc, then we can uncalc_crc
      
      	return crc_table, index_table
      
      
      def calcCRC(inputstr, crc_table):
      	iLen = len(inputstr)
      	crc = 0
      	for i in range(0, iLen):
      		index = (inputstr[i] ^ crc) & 0xff
      		crc >>= 8
      		crc ^= crc_table[index]
      	return crc
      
      
      def unCalcCRC(inputstr, rescrc, crc_table, index_table):
      	finalcrc = rescrc
      	iLen = len(inputstr)
      	i = iLen - 1
      	while(i >= 0):
      		index = index_table[finalcrc >> 56]
      		finalcrc ^= crc_table[index]
      		finalcrc <<= 8
      		finalcrc |= index ^ inputstr[i]
      		i -= 1
      	return finalcrc
      
      
      def calcFlagFormer(welcome_array,crc_table, tar_crc):
      	calcFlagFormerArray = []
      	for i in range(32, 127):
      		print ("[+]in former ", i - 31)
      		for j in range(32, 127):
      			for m in range(32, 127):
      				for n in range(32, 127):
      					welcome_array[0x00] = i;
      					welcome_array[0x10] = j;
      					welcome_array[0x20] = m;
      					welcome_array[0x30] = n;
      					crc = calcCRC(welcome_array, crc_table)
      					if crc == tar_crc:
      						flagarray.append(i)
      						flagarray.append(j)
      						flagarray.append(m)
      						flagarray.append(n)
      						return calcFlagFormerArray
      					if tar_crc == 0:
      						calcFlagFormerArray.append(crc)
      
      	return calcFlagFormerArray
      
      
      def calcFlagLatter(welcome_array, rescrc, crc_table, index_table, tar_crc):
      	calcFlagLatterArray = []
      	for i in range(32, 127):
      		print ("[+]in latter ", i - 31)
      		for j in range(32, 127):
      			for m in range(32, 127):
      				for n in range(32, 127):
      					welcome_array[0x00] = i;
      					welcome_array[0x10] = j;
      					welcome_array[0x20] = m;
      					welcome_array[0x30] = n;
      					crc = unCalcCRC(welcome_array, rescrc, crc_table, index_table)
      					if crc == tar_crc:
      						flagarray.append(i)
      						flagarray.append(j)
      						flagarray.append(m)
      						flagarray.append(n)
      						return calcFlagLatterArray
      					if tar_crc == 0:
      						calcFlagLatterArray.append(crc)
      						
      	return calcFlagLatterArray
      
      def writeToFile(a, filepath):
      	fw = open(filepath, "w")
      	for item in a:
      		fw.write(str(item)+"\n")
      	fw.close()
      
      dividend = 0xC96C5795D7870F42
      (crc_table, index_table) = initCRCtable(dividend)
      #-----------------
      
      tmpstr = '_[Hello___Welcome To Reversing.Kr]__The idea of the algorithm came out of the codeengn challenge__This algorithm very FXCK__But you can solve it!!__Impossible is Impossible_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_['
      welcome_array = []
      for i in tmpstr:
      	welcome_array.append(ord(i))
      welcome_array.append(0) #here is a WTF——for 1h time wasted
      
      tmpReplace = 0x2E9013A335DE51E7;
      for i in range(208, 216):
      	tmp = tmpReplace & 0xff
      	welcome_array[i] = tmp
      	tmpReplace >>= 8
      '''
      a1 = calcFlagFormer(welcome_array[0:64],crc_table, 0)
      print ("[+]calcFlagFormer over.")
      a1.sort()
      print ("[+]FormerCrc sort over.")
      writeToFile(a1, "c:\\users\\chris\\desktop\\mycrc1.txt")
      print ("[+]FormerCrc write over.")
      del a1[:]
      '''
      
      rescrc = 0x676F5F675F695F6C
      rescrc = unCalcCRC(welcome_array[128:], rescrc, crc_table, index_table)
      a2 = calcFlagLatter(welcome_array[64:128], rescrc, crc_table, index_table, 0)
      print ("[+]calcFlagLatter over.")
      a2.sort()
      print ("[+]LatterCrc sort over.")
      
      fr = open("c:\\users\\chris\\desktop\\mycrc1.txt", "r")
      
      tmpa1 = fr.readline()
      a2count = 0
      while(len(tmpa1) > 2):
      	tmp = int(tmpa1[:len(tmpa1) - 1])
      	while (tmp > a2[a2count]):
      		a2count += 1
      	while (tmp < a2[a2count]):
      		tmpa1 = fr.readline()
      		tmp = int(tmpa1[:len(tmpa1) - 1])
      	if tmp == a2[a2count]:
      		break
      
      tar_crc = a2[a2count]
      del a2[:]
      
      flagarray = []
      calcFlagFormer(welcome_array[0:64],crc_table, tar_crc)
      calcFlagLatter(welcome_array[64:128], rescrc, crc_table, index_table, tar_crc)
      
      flagstr = ''
      for i in flagarray:
      	flagstr += chr(i)
      print (flagstr)
      ```

## 24.2 小结

- 很纯粹的CRC的应用，如果会CRC的话会感觉十分容易。*（如果不会，那怨谁，谁教你不会呢）*

## 24.3 后记

一道crpyto的逆向题，应用到了密码学和编码学的知识，没有其他的弯弯绕，就是刚算法的硬题了。

这个题拖了很久，一个是最近忙着打KCTF，一个是需要恶补CRC，一个是程序调了一段时间才跑出正确数据，一个是其他事情也比较多......尤其是历史遗留问题，尽管有心理准备，但打击还是不小。(2019-6-11)

也许是正处于量变到质变的节点，这两年的成长就连我自己都有明显的感觉。删了之前的牢骚，什么都不用说了，不管未来通向何方，都会勇敢、坚定地走下去（2020-8-30）

# 25 CRC2

使用到了CRC算法的线性特征以及掩码的思想。

## 25.1 解题过程

1. 拖进IDA，搜索字符串，定位到401490函数中。F5后会发现很多奇怪的JUMP，查看汇编指令，发现这些JUMP都对应着如下代码：

      ```
      push    33h
      call    $+5
      add     dword ptr [esp], 5
      retf
      ..
      ..
      call    $+5
      mov     dword ptr [esp+4], 23h
      add     dword ptr [esp], 0Dh
      retf
      ```

      使用OD动态运行，发现这些代码没有什么用处，相当于一堆NOP。*（后来参看DoubleLabyrinth大佬的wp才知道这些是WOW64环境逃逸用的）*

      使用IDAPython patch掉这些代码，脚本如下：

      ```python
      import idaapi
      import ida_bytes
      
      ea = 0x401000
      ed = 0x4128e4
      isinit = 0
      while(1):
      	if ea >= ed:
      		break
      	if Byte(ea) == 0xE8 and Byte(ea + 1) == 0x0 and Byte(ea + 2) == 0x0 and Byte(ea + 3) == 0x0 and Byte(ea + 4) == 0x0:
      		isinit = 1
      		x = 0x90
      		if Byte(ea - 2) == 0x6A and Byte(ea - 1) == 0x33:
      			ida_bytes.put_bytes(ea - 2, chr(x))
      			ida_bytes.put_bytes(ea - 1, chr(x))
      		ida_bytes.put_bytes(ea, chr(x))
      		ida_bytes.put_bytes(ea + 1, chr(x))
      		ida_bytes.put_bytes(ea + 2, chr(x))
      		ida_bytes.put_bytes(ea + 3, chr(x))
      		ida_bytes.put_bytes(ea + 4, chr(x))
      		ea += 5
      	elif Byte(ea) == 0xCB and isinit == 1:
      		isinit = 0
      		x = 0x90
      		ida_bytes.put_bytes(ea, chr(x))
      		ea += 1
      	else:
      		if isinit == 1:
      			x = 0x90
      			ida_bytes.put_bytes(ea, chr(x))
      		ea += 1
      ```

2. 此时重新将401490识别为函数，F5查看，就可以得到较为清晰的逻辑：

      ```c
      void __cdecl sub_401490(int a1, int a2, int a3, int a4, int a5, int a6, char a7)
      {
        //...
        //1. 接收输入，要求输入为13个字符
        scanf_401740("%s", InputStr, 20);
        if ( strlen(InputStr) == 13 )
        {
          //2. 初始化byte_410F00
          sub_401000();
          //...
              //3. 检查输入字符
              if ( (v11 < '0' || v11 > '9') && (v11 < 'A' || v11 > 'Z') && (v11 < 'a' || v11 > 'z') )
                break;      
              v8 += v11;
              //...
            		if(v8 == 0x4D0){
                      //一堆计算
                      //4. 计算结果检查
                      if ( v17 == 0x81BAD8907DE045EBi64 )
                        strcpy(aWrong, "Correct\n");
      ```

	1. 用户需要输入一个长为13Byte的字符串，要求仅由字母和数字组成，且累计和为0x4D0
	2. 将用户输入的字符嵌入到byte_410F00中，即byte_410F00[i * 20] = input[i]。计算byte_410F00的CRC值，要求结果为0x81BAD8907DE045EB。

3. sub_401000函数

      这个是CRC的初始化函数，一共初始化了两个数据byte_410F00和dword_411020。其中，byte_410F00是CRC的输入数据，dword_411020则是CRC的查询表。

4. 破解分析

      未知字符有13个，显然不能遍历爆破，即使像CRC1那样拆分也难以承受，因此需要其他方法。

      已知CRC算法具有线性特征——CRC(x) xor CRC(y) == CRC(x xor y)。将初始化的byte_410F00看做y，将嵌入了input的byte_410F00看做x xor y。其中x可以按字符继续拆分为a0 xor a1 xor a2 xor a3...xor a12，ai表示a[i * 20]为输入字符，其余位置为0。每个ai有26*2+10中可能。

      则原式可以变为

      $CRC(x xor y)  xor CRC(y) = CRC(a0) xor CRC(a1) xor CRC(a2)... xor CRC(a12)$

      ai可以按比特继续拆分为b0 xor b1 xor...，bj表示ai的第i个字节的第j个比特。ai = bi.0 xor bi.1... xor bi.7。因此，上式可转变为：

      $CRC(x xor y)  xor CRC(y) = \\CRC(b0.0) xor CRC(b0.1) xor CRC(b0.2)... xor CRC(b0.7)\\xor CRC(b1.0) xor CRC(b1.1) xor CRC(b1.2)... xor CRC(b1.7)\\ ... \\xor CRC(b12.0) xor CRC(b12.1) xor CRC(b12.2)... xor CRC(b12.7)$

      用矩阵来表示，则可得

      ai = [0...1000000...0; 0...0100000...0; 0...0010000...0; 0...0001000...0; 0...0000100...0; 0...0000010...0; 0...0000001...0] * [b0;b1;b2;b3;b4;b5;b6;b7] 

      用ci表示[0...1000000...0; 0...0100000...0; 0...0010000...0; 0...0001000...0; 0...0000100...0; 0...0000010...0; 0...0000001...0] 

      所以x = [c0;c1;...; c12] * [b00; b01; b02; ... b07; b10; b11; ...; b12.7]，

      CRC(x xor y)  xor CRC(y) = [CRC(c0);...; CRC(c12)] *  [b00; b01; b02; ... b07; b10; b11; ...; b12.7]

      其中，CRC(x xor y)  xor CRC(y)和 [CRC(c0);...; CRC(c12)]都已知，所以关键就是求解 [b00; b01; b02; ... b07; b10; b11; ...; b12.7]。*（可以直接参考DoubleLabyrinth大佬的wp，比我讲的清楚多了）*

      解线性方程时需要注意一共有91个变量，64个方程，所以没有唯一解，解由特解和通解组成。

5. 因为是GF(2)上的解，所以对于每个通解来说，系数只有0和1两个选择。一共27个通解，所以一共有2^27个解，可以爆破计算出Flag.

  1. 获取byte_410F00初始化后的数据，脚本如下。需要注意的是，从401490中可以看出byte_410F00大小为256Bytes，但脚本得到的仅有222个字节，余下的0需要自行补全。

         ```python
         import idaapi
         import ida_bytes
         
         ea = 0x40FEC8
         res = []
         for i in range(0, 0x120):
             if Byte(ea + i) != 0:
                 res.append(hex(Byte(ea + i)))
         
                 print res
                 print len(res)
         ```

  2. CRC代码使用破解CRC1时的代码，将初始dividend修改为401000函数中的0x8C997D55124358A1。

         这个脚本分为两部分，一个CRC部分的计算，一个方程求解。
        
         ```python
         #本代码使用时可能需要删掉中文注释
         def initCRCtable(dividend):
         	crc_table = []
         
         	for i in range(0, 256):
         		crc = i
         		for j in range(0, 8):
         			tmp = crc & 0x1
         			crc >>= 1
         			if tmp == 1:
         				crc ^^= dividend
         		crc_table.append(crc) #for improve crc_calc rapid
         
         	return crc_table


         
         def calcCRC(inputstr, crc_table):
         	iLen = len(inputstr)
         	crc = 0
         	for i in range(0, iLen):
         		index = (inputstr[i] ^^ crc) & 0xff
         		crc >>= 8
         		crc ^^= crc_table[index]
         	return crc
         
         #------------------------------------------------------
         
         dividend = 0x8C997D55124358A1
         
         byte_410F00 = [66, 77, 150, 2, 54, 40, 50, 4, 1, 24, 96, 2, 231, 191, 200, 200, 224, 221, 242, 255, 200, 224, 221, 36, 28, 237, 200, 224, 221, 204, 72, 63, 232, 162, 87, 122, 185, 200, 224, 221, 200, 224, 221, 29, 230, 181, 204, 72, 63, 200, 224, 221, 200, 224, 221, 76, 177, 34, 127, 127, 127, 200, 224, 221, 200, 224, 221, 200, 224, 221, 242, 255, 76, 177, 34, 200, 224, 221, 200, 224, 221, 204, 72, 63, 200, 224, 221, 200, 224, 221, 127, 127, 127, 36, 28, 237, 204, 72, 63, 200, 224, 221, 200, 224, 221, 36, 28, 237, 39, 127, 255, 200, 224, 221, 200, 224, 221, 29, 230, 181, 200, 224, 221, 200, 224, 221, 154, 118, 131, 200, 224, 221, 87, 122, 185, 200, 224, 221, 200, 224, 221, 76, 177, 34, 232, 162, 36, 28, 237, 201, 174, 255, 36, 28, 237, 204, 72, 63, 200, 224, 221, 76, 177, 34, 200, 224, 221, 200, 224, 221, 200, 224, 221, 200, 224, 221, 200, 224, 221, 36, 28, 237, 200, 224, 221, 39, 127, 255, 200, 224, 221, 200, 224, 221, 200, 224, 221, 201, 174, 255, 200, 224, 221, 200, 224, 221, 200, 224, 221, 87, 122, 185, 200, 224, 221, 36, 28, 237, 200, 224, 221, 200]
         for i in range(256 - len(byte_410F00)):
         	byte_410F00.append(0)
         
         #这里之所以将inputchr嵌入位置改为0x30而不是使用原始数据，是原始数据中有一些值超过了127。（这个坑调了3个小时才反应过来...）
         #因为要求输入的是ascii码，数值范围在0-127，而且之后各种计算时也都默认最高位为0，所以这里要修改原数据，修改的值可以为0-127内任意数值。
         for i in range(13):
             byte_410f00[i*20] = 0x30
         
         crc_table = initCRCtable(dividend)
         tar_crc = calcCRC(byte_410F00, crc_table) ^^ 0x81BAD8907DE045EB #CRC(y) xor CRC(x xor y)
         
         #[CRC(c0);...; CRC(c12)]
         mask_crc = []
         for i in range(0, 13):
         	index = i * 20;
         	for j in range(0, 7):
         		t = []
         		tmp = 1 << j
         		for z in range(0, 256):
         			if z == index:
         				t.append(tmp)
         			else:
         				t.append(0)
         		crc = calcCRC(t, crc_table)
         		mask_crc.append(crc)
         
         #sage部分参考了DoubleLabyrinth大佬的wp====================================================
         def PrintBitArray2ByteArray(x):
         	res = []
         	for i in range(0, 13):
         		t = 0
         		for j in range(0, 7):
         			t += int(x[i * 7 + j, 0]) * (1 << j)
         		res.append(t)
         	print (res)
         
         def PrintBitTuple2ByteArray(x):
         	res = []
         	for i in range(0, 13):
         		t = 0
         		for j in range(0, 7):
         			t += int(x[i * 7 + j]) * (1 << j)
         		res.append(t)
         	print (res)


         
         a = MatrixSpace(GF(2), 64, 13 * 7)()
         b = MatrixSpace(GF(2), 64, 1)()


         
         for i in range(0, 91):
         	tmp = mask_crc[i]
         	for j in range(0, 64):
         		a[j, i] = (tmp & 0x1)
         		tmp = mask_crc[i] >> (j + 1)
         
         for i in range(0, 64):
         	t = (tar_crc >> i) & 0x1
         	b[i, 0] = t
         
         X = a.solve_right(b)
         
         print "particular solution : "
         PrintBitArray2ByteArray(X)
         
         print "general solution : "
         for base in a.right_kernel().basis():
         	PrintBitTuple2ByteArray(base)
         ```

  3. 根据求解结果，计算可能的flag字符串。

         ```python
         def PrintArray2Str(a):
         	n = 0
         	for i in range(0, len(a)):
         		if(a[i] not in range(ord('0'), ord('9') + 1) and a[i] not in range(ord('a'), ord('z') + 1) and a[i] not in range(ord('A'), ord('Z') + 1)):
         			return
         		n += a[i]
         	if(n != 0x4d0):
         		return
         
         	f = ''
         	for i in range(0, len(a)):
         		f += chr(a[i])
         	print(f)
         
         def ArrayXOR(a, b):
         	res = []
         	for i in range(0, len(a)):
         		res.append((a[i] ^ b[i]))
         	return res
         
         def CalcInputStr(inputstr_origin, parti_solution, general_solution):
         	origin = ArrayXOR(parti_solution, inputstr_origin)
         
         	for i in range(0, 2 ** 27):
         		if(i % 1000000 == 0):
         			print ("here:",(i / 2 ** 27)*100, "%")
         		res = origin
         
         		for j in range(0, 27):
         			if(i & (2 ** j)):
         				res = ArrayXOR(general_solution[j], res)
         
         		PrintArray2Str(res);


         
         byte_410F00 = [66, 77, 150, 2, 54, 40, 50, 4, 1, 24, 96, 2, 231, 191, 200, 200, 224, 221, 242, 255, 200, 224, 221, 36, 28, 237, 200, 224, 221, 204, 72, 63, 232, 162, 87, 122, 185, 200, 224, 221, 200, 224, 221, 29, 230, 181, 204, 72, 63, 200, 224, 221, 200, 224, 221, 76, 177, 34, 127, 127, 127, 200, 224, 221, 200, 224, 221, 200, 224, 221, 242, 255, 76, 177, 34, 200, 224, 221, 200, 224, 221, 204, 72, 63, 200, 224, 221, 200, 224, 221, 127, 127, 127, 36, 28, 237, 204, 72, 63, 200, 224, 221, 200, 224, 221, 36, 28, 237, 39, 127, 255, 200, 224, 221, 200, 224, 221, 29, 230, 181, 200, 224, 221, 200, 224, 221, 154, 118, 131, 200, 224, 221, 87, 122, 185, 200, 224, 221, 200, 224, 221, 76, 177, 34, 232, 162, 36, 28, 237, 201, 174, 255, 36, 28, 237, 204, 72, 63, 200, 224, 221, 76, 177, 34, 200, 224, 221, 200, 224, 221, 200, 224, 221, 200, 224, 221, 200, 224, 221, 36, 28, 237, 200, 224, 221, 39, 127, 255, 200, 224, 221, 200, 224, 221, 200, 224, 221, 201, 174, 255, 200, 224, 221, 200, 224, 221, 200, 224, 221, 87, 122, 185, 200, 224, 221, 36, 28, 237, 200, 224, 221, 200]
         for i in range(256 - len(byte_410F00)):
         	byte_410F00.append(0)
         
         inputstr_origin = []
         for i in range(0, 13):
         	inputstr_origin.append(0x30)
         
         parti_solution = [116, 12, 63, 9, 57, 3, 54, 119, 41, 0, 0, 0, 0]
         
         general_solution = [[1, 0, 0, 0, 52, 121, 16, 125, 29, 55, 54, 4, 99], [2, 0, 0, 0, 70, 101, 4, 60, 5, 71, 5, 40, 11], [4, 0, 0, 0, 70, 121, 63, 27, 2, 77, 44, 29, 50], [8, 0, 0, 0, 70, 65, 73, 85, 12, 89, 126, 119, 64], [16, 0, 0, 0, 9, 118, 114, 106, 51, 6, 42, 0, 22], [32, 0, 0, 0, 121, 8, 65, 27, 20, 83, 99, 5, 10], [64, 0, 0, 64, 126, 66, 93, 22, 35, 15, 44, 111, 73], [0, 1, 0, 0, 49, 27, 33, 48, 25, 94, 29, 62, 2], [0, 2, 0, 64, 41, 87, 121, 47, 86, 18, 122, 49, 63], [0, 4, 0, 64, 31, 124, 12, 91, 102, 111, 25, 82, 90], [0, 8, 0, 0, 119, 12, 10, 95, 65, 76, 111, 122, 25], [0, 16, 0, 0, 89, 24, 88, 98, 119, 60, 93, 90, 115], [0, 32, 0, 64, 11, 34, 70, 94, 50, 4, 39, 114, 119], [0, 64, 0, 0, 68, 6, 34, 107, 38, 93, 104, 69, 61], [0, 0, 1, 64, 90, 18, 44, 42, 126, 48, 50, 68, 21], [0, 0, 2, 64, 84, 6, 36, 64, 107, 74, 33, 23, 88], [0, 0, 4, 0, 71, 27, 77, 99, 40, 95, 44, 91, 4], [0, 0, 8, 64, 8, 80, 19, 76, 53, 22, 28, 85, 12], [0, 0, 16, 64, 74, 99, 116, 30, 72, 93, 23, 2, 22], [0, 0, 32, 64, 114, 22, 99, 125, 105, 87, 25, 6, 44], [0, 0, 64, 0, 78, 72, 66, 16, 97, 19, 2, 92, 7], [0, 0, 0, 65, 97, 34, 89, 3, 58, 95, 78, 62, 121], [0, 0, 0, 66, 29, 115, 0, 64, 125, 123, 101, 58, 106], [0, 0, 0, 4, 120, 96, 54, 48, 126, 103, 105, 82, 73], [0, 0, 0, 8, 105, 87, 4, 122, 54, 67, 56, 42, 30], [0, 0, 0, 16, 25, 91, 77, 100, 3, 71, 6, 103, 113], [0, 0, 0, 96, 87, 33, 52, 3, 122, 50, 64, 95, 107]]
         
         CalcInputStr(inputstr_origin, parti_solution, general_solution)
         ```
        
         结果会输出一堆字符串，其中有一个看着像一句话，这个就是Flag了。

## 25.2 小结

解题关键在于CRC算法线性特征的利用，引入了掩码的思想，还考察了线性方程求解的知识。

在编写sage代码时需要注意与python的区别：

- ^在python中表示异或，在sage中表示乘方。sage中使用^^表示异或。([sage官方文档](http://doc.sagemath.org/html/en/faq/faq-usage.html))
- 二维数组python中下标使用[i\][j]，sage中使用[i, j]

## 25.3 感想

第一次切实意识到线性代数的应用意义，也是第一次使用sage，实在强大。

## 25.4 大佬WP

[https://github.com/DoubleLabyrinth/reversing.kr/tree/master/CRC2](https://github.com/DoubleLabyrinth/reversing.kr/tree/master/CRC2)



---

*（以前觉得要为每个题目单独写一篇博客，时过境迁，现在觉现似乎合起来更好，所以就把以前分散的wp合到了这里。发布时间就设为写第1题wp的时间，更新时间就设为写第25题wp的时间。——2020-8-30）*                                                                                                                                                                                                                                                                                                                        