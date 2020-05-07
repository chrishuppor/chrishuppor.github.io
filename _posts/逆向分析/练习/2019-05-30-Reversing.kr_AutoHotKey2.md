---
layout: post
title: "Reversing.kr_AutoHotKey2"
pubtime: 2019-5-30
updatetime: 2019-5-30
categories: Reverse
tags: WriteUp
---

Reversing.kr的AutoHotKey2，当年绕过的坎总要再走一遍。

# Reversing.kr_AutoHotKey2

## 解题过程

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

      * 第一个DWORD——指出v19的偏移，已知应为0x32800，因此文件中数据为00 28 03 00
      * 第二个DWORD——通过v23的校验，应为v11^0xAAAAAAAA，可以通过OD从内存中获取相应数据。

      需要注意的是，v11的值与除最后四字节数据的全部文件数据相关，所以要先修改第一个DWORD数据，然后再从OD中获取v23的正确值。

4. 将未解压的程序拖进OD，UPX的壳可以使用OD的自解压功能自动脱壳。

   通过IDA可以知道v23的校验代码地址为0x4509DD。OD自解压完成后在该处下断点直接运行，可以得到v23应为0x362BC633。因此，最后四个字节为33 C6 2B 36。

5. 将文件修改好，直接运行得到如下信息。Flag就是这段话描述的人的名字，而且是小写不带空格的。*（权游爱好者实锤了）*

   ![图2 Flag信息](https://chrishuppor.github.io/image/Snipaste_2019-05-30_09-30-11.PNG)

## 启示

这个题没有过多的启示，主要是动静结合的分析以及见招拆招，考验做题人头脑的灵活性。

* 在reverse中，如果某部分算法比较复杂，尤其是难以求逆的那种，无论是逻辑还是内部数值，大概率是没有问题的。一般需要破解的是简单算法的求逆和复杂逻辑算法的参数。