---
layout: post
title: "Reversing.kr_Multiplicative"
date: 2019-6-6 15:37:42
categories: WriteUp
tags: Reversing_kr
---

Reversing.kr的Multiplicative，一个依赖于工具的简单的java逆向题。


# 解题过程

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

# 工具下载

* [jadx下载地址](https://www.softpedia.com/get/Programming/Other-Programming-Files/Jadx.shtml)

# 小结

* python中的数学运算符号

  * python中" / "就表示 浮点数除法，返回浮点结果，" // "表示整数除法。本题应用'//'.
  * python中"*"就表示 乘法，“**”表示乘方。

* python中的有符号整数和无符号整数*（PS.已经被坑过两次了）*

  * 将有符号整数转换为无符号整数

    * 例如将32位的x转换为无符号数——(x & 0xffffffff)

  * 将无符号整数转换为有符号整数

    * 例如将32位的x转换为有符号数

      ```python
      if x > 2 ** 31:
          return (x-2**32)
      else:
          return x
      ```

* 工欲善其事必先利其器。同种工具有很多，这个不行换那个。