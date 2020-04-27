---
layout: post
title: "Reversing.kr_SimpleVM"
pubtime: 2019-6-8 23:52:51
updatetime: 2019-6-8 23:52:51
categories: Reverse
tags: WriteUp
---

Reversing.kr的SimpleVM，一个加了VM壳的ELF程序，不是很simple，用到了pin和IDA远程调试。


# 解题过程

1. 运行程序，提示Access Denied。使用root账户运行程序，发现可以正常运行。所以这个程序要求root权限。

   运行后提示用户输入，如果输入错误则返回Wrong。

2. 程序直接拖进IDA，发现无法解析，果然是加了壳。根据题目名称可知加了VM壳。

3. 黔驴技穷了，只好去看大佬的WP，看懂了再复现一遍。

# 复现过程

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

      * 父进程接收用户输入时限制输入长度小于9.

        ```c
        input = 0;
        v11 = 0;
        v12 = 0;
        read_8048400(0, (int)&input, 10);       // read from stdin,maxbytes is 10
        if ( (_BYTE)v12 )//这里要求输入不能覆盖v12，也就是长度小于9.
        ```

      * 子进程调用sub_8048C6D判断输入是否正确。该函数虽然比较复杂，但是也是可以看的。（参考WP [Reversing.kr题目之SimpleVM详解](https://www.freebuf.com/news/164664.html)）。不过我选择了更省力的方法——pintools

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

# 参考文档

* [windows下使用IDA远程调试linux(ubuntu)下编译的程序](https://blog.csdn.net/lacoucou/article/details/71079552)

# 参考WP

* [Reversing.kr题目之SimpleVM详解](https://www.freebuf.com/news/164664.html)
* [171012 逆向-Reversing.kr（SimpleVM)](https://blog.csdn.net/whklhhhh/article/details/78221365)
* [invicsfate-Reversing.kr](http://invicsfate.cc/2017/09/18/reversing-kr/?nsukey=7EFkQUzSz2nFYRDmRCeolHMtQnmzZBoGYHUU50QI7Bc0xMkHYUIBvpOYNYXJDfrzX4pfToRQKC0iR9Vuzb0y42tJYAvOAmpKODlZ82UZKKYrY639VbLrXMa69bX2Ycyb%2FNlQkylQ23gn2rpzL2wTMyn0lEwegX2xrByI4fkkIwv9u3aZsbMgcRC2D9XNX1iPdo5DrhoVlI%2BFbh9S9xCs0A%3D%3D)

# 小结

* IDA远程调试需要在远程启动相应的server，这个server文件在IDA目录的dbgsrv文件夹中。
* linux下pin安装要对应好版本（gcc版本和系统位数），否则在编译时就是一堆堆的err。以下是我的一些经验：
  * 在ubuntu 18.04 64位机器上安装pin-3.7，可以编译目标平台为intel64的程序，但是ia32的程序就各种报错。
  * 在ubuntu 16.04 32位机器上安装pin-2.7，因为gcc版本不匹配报错；安装pin-3.7，十分舒适。

# 其他

[[脚本及IDB文件下载](https://github.com/chrishuppor/attachToBlog/tree/master/SimpleVM.idb)]