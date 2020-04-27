---
layout: post
title: "Reversing.kr_x64Lotto"
pubtime: 2019-5-29 14:34:9
updatetime: 2019-5-29 14:34:9
categories: Reverse
tags: WriteUp
---

Reversing.kr的x64 Lotto，一个简单的多种解法的x64的题目。

# Reversing.kr_x64Lotto

## 解题过程

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

## 小结

* 这个题有多种解法，每种解法难度不同，其中自己计算v25的数据难度最大（也最蠢），主要考验逆向的灵活度。

* 查看他人的writeup，学到了一个更好的解法。

  * 原理

    当srand()参数相同时，rand()函数所用种子相同，进而生成的伪随机数序列相同。

    在同一秒内，调用time64(0)得到的值是一样的，因此只要输入时使用的伪随机数与程序自身生成随机数生成时间间隔小于1秒就能通过校验。

  * 做法

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
