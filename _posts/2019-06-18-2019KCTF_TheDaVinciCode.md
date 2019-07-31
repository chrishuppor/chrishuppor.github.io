---
layout: post
title: "2019KCTF_TheDaVinciCode"
date: 2019-06-18 8:8:8
categories: WriteUp
tags: CTF
---

2018KCTF的TheDaVinciCode，一个需要应手工具的RE题。


# 解题过程

1. 首先运行程序，熟悉程序流程。本程序是要求用户在文本框中输入数据，点击check按钮，如果没有通过校验则弹出带有"Wrong!"字符串的弹框。

2. 将程序拖入IDA，查找字符串，没有找到Wrong字符串。运行程序，使用process excplorer查看程序内存中的字符串，发现有Wrong字符串。说明这个程序会在内存中解密出要使用的字符串，因此可以借助OD查找Wrong字符串引用地址。

3. 运行程序，使用OD附加该进程。在主模块中查看字符串，果然找到了Wrong字符串。

   如图，可以计算出该字符引用偏移为0x1FDE。

   ![图1 ](https://chrishuppor.github.io/image/Snipaste_2019-06-17_23-29-17.PNG)

4. 将程序拖入IDA，转到偏移0x1FDE处，按F5反编译该函数。该函数为sub_401EA0，简单看一下会发现，这个就是check的关键函数，函数逻辑如下：

   1. 从输入框就收用户输入。用RH查就知道本程序中ID为1000的控件是文本编辑框。

      ```c
      CWnd::GetDlgItemTextW(v1, 1000, &input_str, 20);
      ```

   2. 然后会对输入进行初步检查

      公式如下，分别要求输入字符串长度为16，输入字符要是ascii码。

      ```c
      wcslen(&input_str) == 16 
      (*(&input_str + v2) & 0xFF00) )
      ```

      之后会将wchar转换为char并存储到另一个字符串中，之后操作的都是char类型的字符串。

      ```c
      *(&tmpszChar + v2) = *((_BYTE *)&input_str + 2 * v2);
      ```

   3. 然后动态解密了一个函数，函数为sub_4010E0，大小为0xD17u。

      ```c
      VirtualProtect(sub_4010E0, 0xD17u, 0x40u, &flOldProtect);
      //...
      qmemcpy(sub_4010E0, byte_5647B8, 0x330u);
      VirtualProtect(sub_4010E0, 0xD17u, flOldProtect, &v8);
      ```

      通过查找byte_5647B8的引用，发现byte_5647B8起始的大小为0x330的数据之前在函数sub_401D80中进行过异或处理。

      ```c
      signed int __thiscall sub_401D80(CDialog *this)
      {
        CDialog *v1; // esi
        unsigned int v2; // eax
      
        v1 = this;
        CDialog::OnInitDialog(this);
        SendMessageW(*((HWND *)v1 + 8), 0x80u, 1u, *((_DWORD *)v1 + 29));
        SendMessageW(*((HWND *)v1 + 8), 0x80u, 0, *((_DWORD *)v1 + 29));
        v2 = 0;
        do
          byte_5647B8[v2++] ^= 0xABu; //关键
        while ( v2 < 0x330 );
        return 1;
      }
      ```

   4. 接下来就是调用解密后的sub_4010E0函数，如果该函数返回1就通过检查。

5. 使用IDAPython脚本解密sub_4010E0函数，脚本代码如下：

   ```python
   import idaapi, ida_bytes
   
   ea = 0x5647B8
   ed = 0x4010e0
   for i in range(0, 0x330):
       data = Byte(ea + i) ^ 0xAB
       chrdata = struct.pack("B",data)
       ida_bytes.put_bytes(ed + i, chrdata)
   ```

6. 反编译解密后的sub_4010E0函数。该函数将用户输入的16个字节的字符串划分为两个8字节的字符串，并把这两个字符串看做大整数进行了计算。具体逻辑如下：

   1. 取前8个字节的数据与 0x646E9881E38C9616异或，结果存储在v38；取后8个字节的数据与 0x4F484DBE81DC0884异或，结果存储在v44;

      ```c
      do
      {
          v2 = *((_BYTE *)&v32 + v1) ^ *(_BYTE *)(pInput + v1 + 8);
          *((_BYTE *)&v38_inputformer8 + v1) = *((_BYTE *)&v30 + v1) ^ *((_BYTE *)&v30 + v1 + pInput - (_DWORD)&v30);
          *((_BYTE *)&v44_inputlater8 + v1++) = v2;
      }
      while ( v1 < 8 );
      ```

   2. 分别对v38和v44进行检查，要求v38和v44的第8个字节不为0，但是v38的第八个字节要小于0x10.

      ```c
      //1. v38第8个字节不为0
      do
      {
          if ( *((_BYTE *)&v38_inputformer8 + v4) )   // (Byte*v38)[7] != 0 => pInput[7] ^ 0x64 != 0
              break;
          --v3;
          --v4;
      }
      while ( v4 >= 0 );
      //2. v44的第8个字节不为0
      do
      {
          if ( *((_BYTE *)&v44_inputlater8 + v5) )  // (Byte*v44)[7] != 0 => pInput[15] ^ 0x4F != 0
              break;
          --v3;
          --v5;
      }
      while ( v5 >= 0 );
      //3. v38的第八个字节要小于0x10.
      if ( v3 == 8 && !(v39 & 0xF0000000) )   
      ```

   3. 分别对v38和v44进行乘方运算，并将v44乘方结果乘上7。（代码太长就不贴了)

   4. 最后将计算两个结果的差值，并将差值与0x8比较，如果相等则返回1.

      ```c
      do
      {
          v27 = *((unsigned __int8 *)&former_v56 + v26) - *((unsigned __int8 *)&v46 + v26) - v25;
          *((_BYTE *)&v51 + v26) = v27;
          if ( v27 < 0 )
              v25 = 1;
          ++v26;
      }
      while ( v26 < 17 );
      if ( !v25 )
      {
          v28 = 0;
          while ( *((_BYTE *)&v51 + v28) == *((_BYTE *)&v30 + v28) )
          {
              if ( ++v28 >= 17 )
                  return 1;
          }
      }
      ```

      综上可知，实际就是要求v38 * v38 = 7 * v44 * v44 + 8，接下来尝试计算满足该公式的整数。

7. 8字节的穷举空间太大，难以爆破。所以满足这个公式的数据必然有其特殊之处，因此先计算出一些满足公式的整数，观察其规律。

   计算脚本如下：

   ```python
   import math
   m = 2 
   while (1):
       if m % 7 == 1 or m % 7 == 6:
           t = m * m
           if (t - 8) % 7 == 0:
               x = int(math.sqrt((t - 8) //7))
               if (x * x * 7) == (t - 8):
                   print ("findit-v38:", hex(m), " v44:",  hex(x))
       m += 0x1
   ```

   计算结果如下：

   ```
   findit-v38: 0x6  v44: 0x2
   findit-v38: 0x5a  v44: 0x22
   findit-v38: 0x59a  v44: 0x21e
   findit-v38: 0x5946  v44: 0x21be
   findit-v38: 0x58ec6  v44: 0x219c2
   findit-v38: 0x58931a  v44: 0x217a62
   findit-v38: 0x583a2da  v44: 0x2158c5e
   ```

   可以发现每一个v38大概都是上一个v38的0x10倍，具体计算其比值如下：

   ```
   v38/last v38: 15.0
   v38/last v38: 15.933333333333334
   v38/last v38: 15.93723849372385
   v38/last v38: 15.937253872407457
   v38/last v38: 15.937253932954452
   v38/last v38: 15.93725393319283
   ```

   发现这个比值越来越趋于稳定，大概在15.9372539和15.9372540之间，因此用其控制遍历空间。脚本代码调整如下：

   ```python
   import math
   
   lastone= 0x58931a
   proportion1 = 15.9372539
   proportion2 = 15.9372540
   
   while(1):
       count = 0
       m = int(lastone * proportion1)
       while (m < int(lastone * proportion2) + 1):
           if m % 7 == 1 or m % 7 == 6:
               t = m * m
               if (t - 8) % 7 == 0:
                   x = int(math.sqrt((t - 8) // 7))
                   if (x * x * 7) == (t - 8):
                       print ("findit-v38:", hex(m), " v44:", hex(x), "\nv38/last v38:", m/lastone)
                       lastone = m
                       break
           m += 0x1
   ```

   运行结果如下:

   ```
   findit-v38: 0x583a2da  v44: 0x2158c5e 
   v38/last v38: 15.93725393319283
   findit-v38: 0x57e19a86  v44: 0x21374b7e 
   v38/last v38: 15.937253933193768
   findit-v38: 0x578960586  v44: 0x2115f2b82 
   v38/last v38: 15.937253933193771
   findit-v38: 0x57317ebdda  v44: 0x20f4bb6ca2 
   v38/last v38: 15.937253933193771
   findit-v38: 0x56d9f55d81a  v44: 0x20d3a579e9e 
   v38/last v38: 15.937253933193771
   findit-v38: 0x5682c3dec3c6  v44: 0x20b2b0be7d3e 
   v38/last v38: 15.937253933193771
   findit-v38: 0x562be9e966446  v44: 0x2091dd1903542 
   v38/last v38: 15.937253933193771
   findit-v38: 0x55d5672587809a  v44: 0x20712a6844d6e2 
   v38/last v38: 15.937253933193771
   ```

   当v38 > 0x55d5672587809a时就很难计算出结果了，查其原因是除法运算和开方运算出了问题，因为整数太大。而0x55d5672587809a之后的那个8个字节但第8字节小于0x10的数就是我们想要的，因此可以进行有针对的计算。计算脚本如下：

   ```python
   def getdiv7(a):
       count = 0
       while(a > 0):
           a -= 378818692265664781682717625943 * 282475249
           count += 1
       a += 378818692265664781682717625943 * 282475249
       count -= 1
       x = a // 7 + 7 ** 44 * count
       return x
   
   def getthelast(a):
       print ("input:",a, hex(a))
       nextint = int(a * 15.937253933193771) - 1
       print ("top:",int(a * 15.937253933194) + 1, hex(int(a * 15.937253933194) + 1), '\n')
       while(nextint < int(a * 15.937253933194) + 1):
           mm = nextint * nextint        
           if (mm - 8) % 7 == 0:
               x = getdiv7(mm - 8)
               n = int(x ** 0.5)
               
               nn = n * n
               if (x < nn):                
                   while(nn > x - 1):
                       if nn == x:
                           print ("here:", hex(nextint), hex(n))
                           return nextint, n
                       n -= 1
                       nn = n * n
               else:
                   while(nn < x + 1):
                       if nn == x:
                           print ("here:", hex(nextint), hex(n))
                           return nextint, n
                       n += 1
                       nn = n * n
                   
           nextint += 1
           
   getthelast(0x55d5672587809a)
   ```

   计算结果如下：

   ```
   here: 0x557f3b3b9e1a55a 0x2050988b2bd38de
   ```

8. 编写脚本计算最终的flag:

   ```python
   def calcflag(a, m):
       t = a ^ m
       outstr = ""
       for i in range(0, 8):
           outstr += chr(t & 0xff)
           t = t >> 8
       return outstr
   
   (m, n) = getthelast(0x55d5672587809a)
   a = 0x646E9881E38C9616
   b = 0x4F484DBE81DC0884
   flag = calcflag(a, m)
   flag += calcflag(b, n)
   print (flag)
   ```

   得到flag：L3mZ2k9aZ0a36DMM

# 小结

本题使用了简单的动态解密字符串隐藏关键函数的位置，然后使用简单的动态解密函数隐藏了关键逻辑，使用的都是最简单的技巧，很容易破解。

本题的关键在于sub_4010E0，需要识别出sub_4010E0的大整数运算算法，得到关键公式并且能够计算出符合要求的整数。

本题的难点在于符合要求的整数的计算。

# 启发

* python也是有大整数计算上限的，本题没有达到乘法的上限，但是达到了除法和开方运算的上限，以至于计算8字节整数时除了严重的偏差。此时需要根据实际题目自行设计出准确的除法和开方计算。
  * 除法：利用除法原理，本题中我采用了减7**45的方式来计算<m\>/7的结果
  * 开方：本题中我用了一个较为笨拙的方式，先计算x = sqrt(n)，然后将x++或x--，直到找到x满足x*x = n。
* 2^64根本不可能穷举爆破，所以一定有规律和技巧在，要么不需要爆破要么可以大幅减小爆破空间。
  * 可以通过网络搜索对应的数字获得一定信息。*（PS.我就是受了奇怪启发，试着计算了一下v38/lastv38，然后就打开了大门）*

# 后记

*看了大佬们的WP，再看看自己这个，感觉看到了一个孤陋寡闻之人，深深的感受到工具的重要性，要多向大佬学习。*

WP链接：

* [<https://bbs.pediy.com/thread-252118.htm>](https://bbs.pediy.com/thread-252118.htm)
* [<https://bbs.pediy.com/thread-252068.htm>](https://bbs.pediy.com/thread-252068.htm)

# 附录一:完整脚本

```python
import math

def getsmallm():
    m = 2
    lastone = 1
    prop = 1.0
    while (1):
        if m > 0xfffff:
            return lastone, prop
        if m % 7 == 1 or m % 7 == 6:
            t = m * m
            if (t - 8) % 7 == 0:
                x = int(math.sqrt((t - 8) //7))
                if (x * x * 7) == (t - 8):
                    print ("findit-v38:", hex(m), " v44:", hex(x), " v38/last v38:", m/lastone)
                    prop = float("%0.8f"%(m/lastone))
                    lastone = m
        m += 0x1

def getlargem(lastone, proportion):
    while(1):
        m = int(lastone * proportion)
        if m > 0xffffffffffffff:
            return lastone
        while (1):
            if m % 7 == 1 or m % 7 == 6:
                t = m * m
                if (t - 8) % 7 == 0:
                    x = int(math.sqrt((t - 8) // 7))
                    if (x * x * 7) == (t - 8):
                        print ("findit-v38:", hex(m), " v44:", hex(x), " v38/last v38:", m/lastone)
                        lastone = m
                        break
            m += 0x1

def getdiv7(a):
    count = 0
    while(a > 0):
        a -= 378818692265664781682717625943 * 282475249
        count += 1
    a += 378818692265664781682717625943 * 282475249
    count -= 1
    x = a // 7 + 7 ** 44 * count
    return x

def getthelast(a):
    nextint = int(a * 15.937253933193771) - 1
    while(nextint < int(a * 15.937253933194) + 1):
        mm = nextint * nextint        
        if (mm - 8) % 7 == 0:
            x = getdiv7(mm - 8)
            n = int(x ** 0.5)
            
            nn = n * n
            if (x < nn):                
                while(nn > x - 1):
                    if nn == x:
                        print ("here:", hex(nextint), hex(n))
                        return nextint, n
                    n -= 1
                    nn = n * n
            else:
                while(nn < x + 1):
                    if nn == x:
                        print ("here:", hex(nextint), hex(n))
                        return nextint, n
                    n += 1
                    nn = n * n
                
        nextint += 1

def calcflag(a, m):
    t = a ^ m
    outstr = ""
    for i in range(0, 8):
        outstr += chr(t & 0xff)
        t = t >> 8
    return outstr
import math

def getsmallm():
    m = 2
    lastone = 1
    prop = 1.0
    while (1):
        if m > 0xfffff:
            return lastone, prop
        if m % 7 == 1 or m % 7 == 6:
            t = m * m
            if (t - 8) % 7 == 0:
                x = int(math.sqrt((t - 8) //7))
                if (x * x * 7) == (t - 8):
                    print ("findit-v38:", hex(m), " v44:", hex(x), " v38/last v38:", m/lastone)
                    prop = float("%0.8f"%(m/lastone))
                    lastone = m
        m += 0x1

def getlargem(lastone, proportion):
    while(1):
        m = int(lastone * proportion)
        if m > 0xfffffffffffff:
            return lastone
        while (1):
            if m % 7 == 1 or m % 7 == 6:
                t = m * m
                if (t - 8) % 7 == 0:
                    x = int(math.sqrt((t - 8) // 7))
                    if (x * x * 7) == (t - 8):
                        print ("findit-v38:", hex(m), " v44:", hex(x), " v38/last v38:", m/lastone)
                        lastone = m
                        break
            m += 0x1

(lastone, proportion) = getsmallm()
lastone = getlargem(lastone, proportion)
(m, n) = getthelast(lastone)
(m, n) = getthelast(m)
a = 0x646E9881E38C9616
b = 0x4F484DBE81DC0884
flag = calcflag(a, m)
flag += calcflag(b, n)
print (flag)
```