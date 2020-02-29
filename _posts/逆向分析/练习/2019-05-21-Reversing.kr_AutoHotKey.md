---
layout: post
title: "Reversing.kr_AutoHotKey"
date: 2019-5-21 22:38:45
categories: Reverse
tags: WriteUp
---

Reversing.kr的第十三题，一个需要静态分析配合动态调试的题目——动态调试方便看内存数据，断点功能也方便根据功能定位函数的调用；静态分析方便看程序逻辑。

# Reversing.kr_AutoHotKey

## 解题过程

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

## 小结

* 逆向题毕竟不是实践，所以其中的算法一般不会很复杂，都是可解的。如果一个算法十分复杂，就要考虑是否可以通过动态运行，利用程序自身进行运算，然后从内存中提取结果。
* 动静结合分析，通过静态分析找逻辑，通过动态分析找位置。找到位置后可以返回静态进行分析。
* 通过查看他人writeup，发现AutoHotKey是一个成熟的开源框架，用于将脚本转换为exe，也可以再从exe将脚本提取出来。在转换时需要一个Password，也就是本题中的DecryptKey。如果要从exe中恢复脚本，当然要求exe不能有变化，所以会需要进行exe的md5验证，只不过作者没有直接计算exe的md5，而是自己设计了一个exe完整性校验算法。如此一来，这个题目的背景就清楚了——为从exe中恢复脚本，需保证程序完整性以及获得转换密钥。
* md5在线解密网址：[http://pmd5.com/](http://pmd5.com/)