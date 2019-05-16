---
layout: post
title: "Reversing.kr_Twist1"
date: 2019-5-16 10:24:21
categories: WriteUp
tags: Reversing_kr
---

Reversing.kr的第十题，题如其名，很考研动态调试的一道题，OD用的熟不熟很关键。

# Reversing.kr_Twist1

## 解题过程

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

         ![Snipaste_2019-05-15_22-26-12.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-26-12.PNG)

         如下图，检查PEB.ProcessHeap.Flag是否为2，也就是检查[[FS:30] + 0x18]+0xC是否是2。（非调试时应该不为2）

         ![Snipaste_2019-05-15_22-27-53.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-27-53.PNG)

         接下来又检查了[[FS:30] + 0x18]+0x40是否为2。（非调试时应该不为2）

         ![Snipaste_2019-05-15_22-32-47.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-32-47.PNG)

         接下来再次获取了PEB.ProcessHeap，并检查[[FS:30] + 0x18]+0x44是否为0.（非调试时不为0）

         ![Snipaste_2019-05-15_22-34-34.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-34-34.PNG)

         接下来惊现与0xABABABAB和0xEEFEEEFE的比较。这个是通过堆中数据检查进行反调中常用的比较。查看EDI的来源，果然是来自于堆空间。此时需要将所检查的空间中的0xABABABAB和0xEEFEEEFE改为其他数据才能通过测试，或者修改跳转逻辑。

         ![Snipaste_2019-05-15_22-36-55.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-36-55.PNG)

         接下来发现注册了一个SEH，将函数403903注册为异常处理函数。

         ![Snipaste_2019-05-15_22-47-57.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-47-57.PNG)

         接下来看到了GetVersion和GetCommand，是不是见到亲人了？没错，这里就是start函数了，之前注册SEH也是start的正常操作。

         ![Snipaste_2019-05-15_22-48-55.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-48-55.PNG)

         接着根据main函数的参数特点结合个人经验就找到main函数入口了，如下。

         ![Snipaste_2019-05-15_22-54-44.PNG](https://chrishuppor.github.io/image/Snipaste_2019-05-15_22-54-44.PNG)

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

            这个部分也不太平，很考验耐心和细心。而且各种跳转，所以就不截图了。

            1. 检查0x40C450是否为0x36，也就是说input[6]^0x36==0x36，所以input[6] == 0

            2. 将Input打乱顺序存储到内存，这些内存地址就是接下来要使用的东西，注意记录对应关系就好，不再细说。

               1. 检查[0x40B000]是否为0x49，而[0x40B000] == input[0] ror 6，所以input[0] == 0x49 rol 6 == 'R'

               2. 检查[0x40CCE0] ^ 0x77是否为0x35，所以input[2] == 'B'

               3. 检查[0x40CCEC]^0x20是否为0x69，所以input[1] == 'I'

               4. 接下来居然又检查了一下input[0]，而且跟第一次的值还不一样。

                  用户的输入只有一个值，在这两次不同的检查之间也没有进行修改，而且确信第一次检查结果要为真，所以这次检查结果为假。这是一个用来迷惑破解者的小把戏。

               5. 检查[0x40C401]^0x21是否为0x64，所以input[3] == 'E'

               6. 检查[0x40CD30]^0x46是否为0x14,。但是之前没有记录0x40CD30对应input的哪一位，不过查看0x40CD30发现是我input的第5个字符，所以应该是input[4]。因此，input[4] == 'R' 

               7. [0x40CCFC] rol 4 == 0x14，所以input[3] == 0x14 ror 4 == 0x65 == 'A'


## 小结

* 这道题一共有三个点，每个点都有一定难度，稍有不慎就要重来。
  * 脱壳找到OEP
  * 绕过所有反调
  * 破解input匹配逻辑
* 直接运行程序是可以的，使用OD加载运行程序失败的——程序极可能有**反调功能**，反调总结请见另一片博客 [静态反调小结]
* 单步调试时注意不要步过call，而是要追进去，要不然哪里崩的都不知道......
* **OD的标签**：如果某个函数将程序导向错误的路径，要使用shift+;为其添加标签，这样在条件跳转时就能清楚的知道如何跳转才是正确的。

*PS:被这个题的反调虐哭，几乎使用了全部的静态反调手段，是个专场......其中大部分在《逆向工程核心原理》里面有讲解。其中，检测HEAP.0x40和HEAP.0x44的值，检测关键函数是否被挂钩，以及使用GetThreadContext检测硬件调试寄存器的方法是我新学到的。真心的感谢作者。*