---
layout: post
title: "Reversing.kr_MetroApp"
date: 2019-06-05 8:8:8
categories: WriteUp
tags: Reversing_kr
---

Reversing.kr的MetroApp，一个我觉得学了也没有用的题目。


# 解题过程

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

# 参考文章

* MetroApp程序安装
  * [【UWP开发】uwp应用安装失败](https://blog.csdn.net/egostudio/article/details/77408126)
  * [如何在Windows 10中手动安装.appx文件](https://www.sysgeek.cn/windows-10-install-appx-uwp/)

# 小结

* string.exe和IDA string功能得到的字符串结果不同。

* 虽然最后找到了Flag，但仍然不知道为什么就能弹出correct。查看了别人的WP，大佬们也没有给出更多的信息，而且基本也是迷迷糊糊的。怎么说呢？感觉逆向除了技术，更多的是靠社工的推理（连蒙带猜）。