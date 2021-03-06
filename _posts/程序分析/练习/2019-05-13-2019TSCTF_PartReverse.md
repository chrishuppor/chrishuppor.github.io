---
layout: post
title: "2019TSCTF_PartReverse"
pubtime: 2019-5-13
updatetime: 2019-5-13
categories: Reverse
tags: WriteUp
---

2019年TSCTF的两道Reverse.

# 1 BlackTea

[[题目及IDB文件下载](https://github.com/chrishuppor/attachToBlog/tree/master/BlackTea)]

## 1.1 解题

* 程序分析

  一个ELF文件，先运行一下，也是输入一个字符串，然后给出提示信息。

  拖进IDA，函数比较少。查看导入表，导入的函数也很少。查看字符串，看到程序的提示信息。如果程序没有做隐藏，那么这个程序破解关键就是算法了。

  定位到Try again!的引用位置，反编译，发现这是main函数，而且十分简单——接收一个输入，然后进行两个判断。

  ```c
  puts("Input your flag:");
    __isoc99_scanf("%s", &input_str);
    if ( (unsigned int)CalcInputToArg2_400686(&input_str, (__int64)&CalcRes) && (unsigned int)sub_4007C4(&CalcRes) )
      puts("Congratulation!");
    else
      puts("Try again!");
  ```

  所以关键在判断的两个函数中。

  * 查看第一个函数

    首先判断输入是否是23个字符。

    ```c
      if ( (unsigned int)strlen(input) != 23 )
        return 0LL;
    ```

    然后将第5个字符以后的字符进行散列替换，替换为其在dword_602480中的位置。

    ```c
      for ( i = 0; i < 23; ++i )
      {
        if ( i > 5 )
        {
          if ( 22 == i )
          {
            v6 = input[i] == '}';
          }
          else
          {
            v3 = v5++;
            *(_BYTE *)(a2 + v3) = dword_602480[input[i]];
          }
        }
        else
        {
          *((_BYTE *)&s2 + i) = input[i];
        }
      }
    ```

    最后比较str_tsct和s2的6个字符是否一致以及v6是否为真，并返回。其中str_tsct硬编码在程序中为"TSCTF{"；v6 = input[i] == '}'，其中i==22，也就是说要求input[22] == ‘}’。

    ```c
      return !memcmp(&str_tsct, &s2, 6uLL) && v6;
    ```

    因此，本函数可以得到的信息是

    * Flag一共23个字符，以'TSCTF{'开始，以'}'结束
    * 对{}之间的数据进行散列变换，变换的表为dword_602480。

  * 查看第二个函数

    首先将第一个函数变换后的结果转存，然后是一个循环。如下，将ResCopy中字符逐一与dword_602080[16 * j + k]相乘再累加，类似于sj = aj0 *x0 + aj1 * x1 + aj2 * x2+...+ aj15 * x15，其中aji为dword_602080[16 * j + i]，xi位ResCopy[i]。

    ```c
    for ( j = 0; j <= 15; ++j )
    {
        for ( k = 0; k <= 15; ++k )
            s1[j] += *(&ResCopy + k) * dword_602080[16 * j + k];
    }
    ```

    接下来又是一个循环，显得有些复杂，应该不是作者自己设计的。联想到程序名称Black_TEA，貌似TEA是个加密方式。在网上搜索TEA算法，与这个循环一致。

    ```c
    for ( l = 0; l <= 7; ++l )
      {
        s1_j1 = s1[2 * l];
        s1_j2 = s1[2 * l + 1];
        int_v8 = 0;
        for ( m = 0; m <= 31; ++m )
        {
          int_v8 -= 0x61C88647;
          s1_j1 += (s1_j2 + int_v8) ^ (16 * s1_j2 + 0x19260817) ^ ((s1_j2 >> 5) + 0x11111111);
          s1_j2 += (s1_j1 + int_v8) ^ (16 * s1_j1 + 0x424255AA) ^ ((s1_j1 >> 5) - 0x5D2CED22);
        }
        s1[2 * l] = s1_j1;
        s1[2 * l + 1] = s1_j2;
      }
    ```

    最后将计算结果与程序中的数据进行比较。

    ```c
    memcmp(s1, &s2, 0x40uLL) == 0
    ```

  至此，Flag的正向操作已经明了：TSCTF{}的标志不变，先对{}中的16个字符进行散列变换，然后进行一个线性计算得到16个int的结果，然后将这16个int进行一个TEA加密。

  那么反向求解就是将上述三个操作逐一逆回去——先TEA解密获得那16个int，然后解一个线性方程组求出16个char，最后根据散列表获得原始的16个char。

* Flag求解

  * TEA解密

    根据TEA加密算法可知，加密过程中- 0x5D2CED22应该为 + 0xa2d312de；int_v8 -= 0x61C88647应该为int_v8 +=0x9e3779b9，32轮循环后int_v8为0x9e3779b9 * 32。（如果不确定自己搞的key是否正确，可以使用gdb动态加载程序，定位到加密时的循环，直接从内存中获取key。）

    解密代码如下

    ```c
    void decrypt(uint32_t* v, uint32_t* k) {
    	uint32_t v0 = v[0], v1 = v[1], sum = 0x9e3779b9 * 32, i;  /* set up */
    	uint32_t delta = 0x9e3779b9;                     /* a key schedule constant */
    	uint32_t k0 = k[0], k1 = k[1], k2 = k[2], k3 = k[3];   /* cache key */
    	for (i = 0; i<32; i++) {                         /* basic cycle start */
    		v1 -= ((v0 << 4) + k2) ^ (v0 + sum) ^ ((v0 >> 5) + k3);
    		v0 -= ((v1 << 4) + k0) ^ (v1 + sum) ^ ((v1 >> 5) + k1);
    		sum -= delta;
    	}                                              /* end cycle */
    	v[0] = v0; 
    	v[1] = v1;
    }
    
    int main()
    {
    	uint32_t v[16] = { 0xDE379917, 0x5BFE915C, 0xF9D4D5CB, 0x9A5DFE7F,
    		0x9FE2930B, 0x56B4E3A0, 0x479638DB, 0x8C01F2AD,
    		0xD15930E8, 0xB9C1C715, 0xFFFF4E67, 0x73846A5F,
    		0x33C9C9B9, 0xBD9CD5B3, 0x2B31EA2B, 0xAEEDFCAD };
    	uint32_t k[4] = { 0x19260817, 0x11111111, 0x424255AA, 0xa2d312de };
    	// v为要加密的数据是两个32位无符号整数  
    	// k为加密解密密钥，为4个32位无符号整数，即密钥长度为128位  
    	for(int i = 0; i < 8; i++){
    		decrypt(&v[i*2], k);	
    		printf("%d,%d,", v[i*2+0], v[i*2+1]);
    	}
    	system("pause");
    	return 0;
    }
    ```

    得到解密结果如下

    ```
    [3143859,8171460,3146919,-5858067,-22399063,-7081089,937345,1669424,8037239,1157740,4334495,-7799960,2513032,20181947,4208148,-5405833]
    ```

  * 线性方程求解

    首先使用IDAPython脚本将方程组系数矩阵提取出来。

    需要注意的是，系数矩阵中有负数，而直接提取的为无符号整数，需要自己处理一下。另外，idapython输出的系数表中会为大整数添加L，需要用全局替换删掉。

    ```python
    import idaapi
    
    print '[',
    for j in range(0,16):
        a = []
        for k in range(0,16):
            tmp = Byte(0x602080 + (16 *j + k)*4 + 3)        
            tmp = tmp * 256 + Byte(0x602080 + (16 *j + k)*4 + 2)
            tmp = tmp * 256 + Byte(0x602080 + (16 *j + k)*4 + 1)
            tmp = tmp * 256 + Byte(0x602080 + (16 *j + k)*4 + 0)
            if (tmp & 0x80000000):
                tmp = (-(-tmp & 0xffffffff))
            a.append(tmp)
        if j != 15:
            print a,','
        else:
            print a,
    print ']'
    ```

    求解方程组线性方程组，脚本如下。需要注意的是，solve求解出的x是一个浮点数数组，需要使用int转换为整数，而且转换前还需要用round四舍五入一下。如果直接使用int向下取整会造成某些数值始终错误的血案。

    ```python
    import numpy as np
    from scipy.linalg import solve
    
    a = np.array([[-3726,51845,41954,36830,-16607,-54360,-3218,-18682,-16023,-3043,-30540,-10451,16218,-21422,33308,64068],
    [39001,51934,42337,23183,-14657,-53881,-2653,-18234,-12854,-12430,-29432,-8445,12057,-21370,33261,59208],
    [-3726,51845,41954,36847,-16607,-54360,-3218,-18682,-16026,-3043,-30536,-10451,16218,-21422,33308,64071],
    [-3726,51137,41963,36599,-13864,-54363,-50119,-18753,-12797,-3253,-30803,-10394,12062,-21426,11279,63062],
    [-4939,-24953,12424,18437,-16585,-58168,-35534,-63482,-3231,-12159,-29883,-11005,8122,-57176,48489,32034],
    [3436,7808,42655,36835,-17159,-54101,-51352,-5565,-12795,-13404,-32541,-20783,52476,-22024,11163,63112],
    [-3734,12043,41948,36877,-16608,-54360,-3261,-18682,-16023,-2986,-30714,-10404,12060,-21424,33307,63204],
    [-2905,51587,41960,36573,-16462,-54183,-3218,-18740,-12923,-3262,810,-10204,16192,-21396,11272,62310],
    [37358,52023,41326,36828,-16711,-54346,-50236,-5334,-12792,-2856,-833,-21279,52821,-21281,33419,43637],
    [5446,51854,14953,28732,36728,-53408,-51684,-63369,-12853,-13662,-36803,-2261,16218,-15903,12295,57855],
    [-3729,51845,41954,36830,-13678,-54360,-3219,-18683,-12925,-3043,-30532,-10454,16189,-21422,33308,64068],
    [1848,12050,14766,20634,8297,-54360,-7554,-3748,-56014,-61174,29752,-3484,43484,-9504,33308,55289],
    [-3726,12051,41954,36847,-13665,-54360,-3218,-18681,-12926,-3043,-30534,-10454,16229,-21422,33308,64071],
    [23732,50626,41374,32790,39511,-53408,-3266,-5795,-13117,-3043,-38154,-20776,55993,-21221,33308,45946],
    [32752,52006,43678,34662,-13271,-53972,-3577,-18679,-13037,-13104,-33148,-17498,11848,-22032,11292,61561],
    [-3707,51851,16268,34652,20074,-52456,-49522,-5564,-12862,-12564,-45768,-6518,59376,-15501,11383,15466]])
    b = np.array([3143859,8171460,3146919,-5858067,-22399063,-7081089,937345,1669424,8037239,1157740,4334495,-7799960,2513032,20181947,4208148,-5405833])
    
    x = solve(a, b)
    res = []
    for i in range (0, 16):
        res.append(int(round(x[i])))
    print (res)
    ```

  * 散列表求解

    首先使用idapython提取dword_602480的散列表。脚本如下：

    ```python
    import idaapi
    a = []
    for i in range(0, 256):
        tmp = Byte(0x602480 + i*4 + 3)        
        tmp = tmp * 0x10 + Byte(0x602480 + i*4+ 2) 
        tmp = tmp * 0x10 + Byte(0x602480 + i*4+ 1) 
        tmp = tmp * 0x10 + Byte(0x602480 + i*4 + 0) 
        a.append(tmp)
    print(a)
    ```

    然后进行比对获得flag（这部分代码是跟在方程组求解后面的）

    ```python
    dword_602480 = [208, 201, 0, 198, 126, 99, 124, 221, 213, 80, 211, 43,
         140, 170, 228, 93, 194, 224, 243, 10, 136, 18, 173,
         250, 189, 130, 135, 210, 122, 22, 127, 159, 157, 94,
         129, 230, 107, 203, 217, 132, 244, 21, 123, 49, 131,
         148, 142, 154, 205, 90, 174, 245, 4, 229, 192, 199,
         226, 59, 232, 42, 164, 81, 106, 98, 74, 116, 216, 78,
         195, 92, 252, 11, 36, 156, 160, 40, 223, 165, 227, 17,
         14, 66, 177, 109, 27, 44, 68, 190, 73, 100, 212, 64,
         28, 147, 105, 220, 23, 191, 254, 6, 96, 51, 75, 84,
         239, 197, 47, 231, 63, 58, 46, 102, 25, 255, 120, 249,
         61, 121, 108, 60, 125, 169, 234, 204, 7, 167, 183, 219,
         233, 26, 39, 56, 222, 188, 185, 76, 71, 101, 151, 246,
         67, 139, 86, 72, 145, 55, 182, 144, 134, 247, 225, 118,
         54, 163, 237, 2, 162, 91, 13, 175, 89, 218, 184, 53,
         117, 248, 110, 97, 12, 242, 9, 48, 95, 158, 19, 166,
         57, 104, 149, 152, 87, 70, 168, 79, 236, 35, 172, 187,
         180, 3, 113, 253, 77, 34, 33, 155, 150, 153, 32, 178,
         112, 235, 119, 202, 251, 181, 30, 114, 62, 37, 146,
         52, 115, 133, 82, 209, 15, 176, 238, 111, 128, 193,
         31, 1, 8, 45, 50, 186, 24, 143, 214, 141, 69, 171, 215,
         65, 41, 138, 206, 20, 38, 88, 137, 161, 83, 16, 196,
         241, 85, 29, 5, 103, 179, 240, 200, 207]
    
    flag = ''
    #res为方程求解结果得到的char数组
    for item in res:
        flag = flag + chr(dword_602480.index(item))
    print (flag)
    ```

    得到”0n1y_MATRiX+t3a.“，所以最终Flag为TSCTF{0n1y_MATRiX+t3a.}

## 1.2 小结

* 这是一个算法类型的逆向题，关键是求逆算法。本题在求逆的时候有三个大坑
  * TEA算法中都是用的加，而IDA识别成了减，因此在构造key的时候注意将负数转换为整数。大概是因为IDA中硬编码的数都被识别成无符号整数，所以加变成了-。
  * 线性方程系数中存在负数，而IDA不识别负数，需要自行处理一下。
  * 解方程结果为float类型，需要转为int才可以进行散列替换。

# 2 CheckIn

[[题目及IDB文件下载](https://github.com/chrishuppor/attachToBlog/tree/master/CheckIn)]

- 程序分析

  是一个ELF程序，拖进IDA，发现只有一个函数。查看Strings，发现没有字符串信息。

  反编译start函数，在开头有一堆赋值操作，转换为字符串，得到字符串如下：

  ```c
    v36 = ' ';
    v35 = ':ega';
    v34 = 'ssem';
    v33 = ' ruo';
    v32 = 'y ev';
    v31 = 'ael ';
    v30 = 'woN\n';
    input_str = '9102';
    v28 = 'FTCS';
    v27 = 'T ot';
    v26 = ' emo';
    v25 = 'cleW';
  ```

  接着是一个循环，将标准输入的字符逐个读入到&v25中，v25中字符循环右移v1位存储到input_str中。其中v1位循环计数，也是input_str和v25的下标。最多循环32次或输入回车结束循环。

  ```c
  do
    {
      v2 = sys_read(1, &v25, 1u);                 // read to v25, one byte once
      if ( (_BYTE)v25 == '\n' )                   // 回车结束
        break;
      *((_BYTE *)&input_str + v1) = __ROR1__(v25, v1);
      ++v1;
    }
    while ( v1 != 32 );
  ```

  接着又有一个循环，显然是在逐字符比对v3和v4，一共比较v5次，如果都匹配则输出成功的字符串。其中v3 = input_str，是上个循环得到的字符串；v4是硬编码在程序中的。

  ```c
  v24 = 0xBE5F;
  v23 = 0xE8C90B45;
  v22 = 0x6ED7996D;
  v21 = 0x801DDB64;
  v20 = 0x8AD0A954;
  v4 = &v20;
  while ( 1 )
    {
      v6 = *(_BYTE *)v3;                          // 要求input = v20
      v7 = *(_BYTE *)v4;
      v3 = (int *)((char *)v3 + 1);
      v4 = (int *)((char *)v4 + 1);
      if ( v6 != v7 )
        break;
      if ( !--v5 )                                // print yes
      {
        v19 = 1;
        LOBYTE(v19) = 0;
        v18 = '\n): ';
        v17 = 'ti e';
        v16 = 'kam ';
        v15 = 'uoy ';
        v14 = ',seY';
        v9 = sys_write(0, &v14, 20u);
        goto LABEL_9;
      }
    }
  ```

  至此，Flag的操作就明确了：经过第一个循环的处理，然后要求结果是&v4所指向的数据。因此只需将&v4所指数据按位置依次循环左移就可以了。

- 解题脚本如下

  ```python
  import time
  import os
  
  v24 = 0xBE5F;
  v23 = 0xE8C90B45;
  v22 = 0x6ED7996D;
  v21 = 0x801DDB64;
  v20 = 0x8AD0A954;
  
  v = [0x54, 0xA9, 0XD0, 0X8A,
       0X64, 0XDB, 0X1D, 0X80,
       0X6D, 0X99, 0XD7, 0X6E,
       0X45, 0X0B, 0XC9, 0XE8,
       0X5F, 0XBE]
  
  flag = ''
  
  i = 0;
  for item in v:
      i = i % 8
      tmpL = (item >> (8-i))
      tmpH = (item<<i) % 0xff
      tmp = tmpH | tmpL
      print(hex(tmpH), hex(tmpL))
      i = i + 1
      flag = flag + chr(tmp)
  
  print(flag)
  ```

# 3 LongCheck

主要考察指令流获取。[[题目及IDB文件下载](https://github.com/chrishuppor/attachToBlog/tree/master)]

## 3.1 解题思路

1. 拖入IDA，程序根本无法看。

2. 拖入OD。通过查找字符串，很容易定位到check函数——sub_offset2C0。其余代码十分简单，对于sub_offset2C0也没有什么好方法，先单步追踪进入到check函数，看看情况。

3. 进入到sub_offset2C0，单步运行，逐渐就会发现check的策略和函数的特点。

   check策略：逐个检查用户输入的字符的比特位。其中，检查的顺序不确定，并且有重复的检查。

   函数的特点：使用了花指令，但check的结构十分清晰——首先将用户输入字符串存储地址赋值给eax，然后通过eax+xxx获取某个字符，通过SHR和AND指令获取字符的某一位，最后使用CMP进行比较，正确则jmp到下一个check。

   check块关键汇编代码如下：

   ```汇编
   mov eax, [ebp - 8]
   mov al, [eax + <x>]
   shr al,<y>
   and al,0x1
   cmp al,<z>
   je <next_check>
   ```

   获取flag的关键信息就是x、y、z——即第x个字符的第y位为z（xy从零开始）。那么如何获取xyz数据呢？

4. 因为花指令的存在，无法使用IDApython直接提取代码，但可以根据机器码将mov、shr、cmp的关键数据提取出来。

   机器码对应如下：

   | 汇编代码           | 机器码     |
   | ------------------ | ---------- |
   | mov eax, [ebp-8]   | 8b 45 F8   |
   | mov al, [eax+<x\>] | 8a 40 <x\> |
   | shr al, <y\>       | d0 e8 <y\> |
   | cmp al, <z\>       | 3c <z\>    |

   需要自行解析je和jmp函数进行跳转:

   - je <x\> = 跳转到（当前地址 + x +2)，je指令为0x74
   - jmp <x\> = 跳转到（当前地址 + x +2)，jmp指令为0xEB

   在运行脚本时发现还有其他地方有0x74和0xEB，需要忽略这些地方：

   - call sub_offset2b0: 有时0x74和0xEB会出现在call指令中，幸运的是check函数中仅有call sub_offset2b0指令出现了0x74和0xEB。因此可以在当发现机器码E8，计算其跳转的地址，如果为offset2b0则步过该指令。
   - mov ebx, 5eb: 该指令机器码为66 bb eb 05，因此当发现eb时，如果之前两个字节为66 bb，则应该步过。但是有时会直接jmp到eb所在地址，此时需要解析为jmp函数。

   check函数中有很多ret指令，本来无法自行解析，但本函数ret指令相当于nop。另外需要注意的是：shr al, 1的指令有其特殊的机器码。

5. 根据以上分析编写脚本获取flag。脚本运行结果如下：

![图1 脚本运行结果](https://chrishuppor.github.io/image/Snipaste_2019-06-12_21-38-43.PNG)

## 3.2 解题脚本

IDAPython脚本如下

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import idaapi

ea = 0xFA1426 #check起始地址
tarcount = 2045 #n次运行后发现一共check了2045次
jmpea = 0 #记录要跳转的地址

flagarray = []
flaglen = 100
for i in range(0, flaglen):
	flagarray.append([0,0,0,0,0,0,0,0])

count = 0;
while(1):
	if count == tarcount:
		break
        
	#mov eax, [esp-8]
	if (0x8b == Byte(ea)) and (0x45 == Byte(ea + 1)) and (0xF8 == Byte(ea + 2)):
		count += 1
		print("[+]in section %d at 0x%x"%(count, ea))
        
		ea += 3

		breakflag = 0;
		item = {"mov":0, "shr":0, "cmp":0}
		while(1):
			#shr al, xxx
			if 0xc0 == Byte(ea) and 0xe8 == Byte(ea + 1):
				item["shr"] = Byte(ea + 2)
				ea += 3

			#shr al,1
			elif 0xd0 == Byte(ea) and 0xe8 == Byte(ea + 1):
				item["shr"] = 1
				ea += 2

			#mov al, xxx
			elif 0x8a == Byte(ea) and 0x40 == Byte(ea + 1):
				item["mov"] = Byte(ea + 2)
				ea += 3

			#cmp al, x
			elif 0x3c == Byte(ea) and (Byte(ea + 1) == 0 or Byte(ea + 1) == 1):
				item["cmp"] = Byte(ea + 1)
				flagarray[item["mov"]][item["shr"]] = item["cmp"]
				ea += 2
				break

			#jmp
			elif (0xEB == Byte(ea) or 0x74 == Byte(ea)):
				#66 bb eb 05并且不是跳转到这里的
				if 0xEB == Byte(ea) and Byte(ea - 1) == 0xbb and Byte(ea - 2) == 0x66 and ea != jmpea:
					ea += 1
                 
                #解析跳转
				else:
					x = Byte(ea + 1)
					if x > 2 ** 7:
						x = x - 2 ** 8

					ea = ea + x + 2
					jmpea = ea

			#call
			elif 0xE8 == Byte(ea):
				x = Byte(ea + 4) << 24 | Byte(ea + 3) << 16 | Byte(ea + 2) << 8 | Byte(ea + 1)
				if x > 2 ** 31:
					x = x - 2 ** 32

				if ea + x +5 == 0xfa12b0:
					ea += 5
				else:
					ea += 1

			else:
				ea += 1

	#jmp
	elif (0xEB == Byte(ea) or 0x74 == Byte(ea)):
		if 0xEB == Byte(ea) and Byte(ea - 1) == 0xbb and Byte(ea - 2) == 0x66 and ea != jmpea:
			ea += 1
		else:
			x = Byte(ea + 1)
			if x > 2 ** 7:
				x = x - 2 ** 8

			ea = ea + x + 2
			jmpea = ea

	elif 0xE8 == Byte(ea):
		x = Byte(ea + 4) << 24 + Byte(ea + 3) << 16 + Byte(ea + 2) << 8 + Byte(ea + 1)
		if x > 2 ** 31:
			x = x - 2 ** 32

		if ea + x +5 == 0xfa12b0:
			ea += 5
		else:
			ea += 1

	else:
		ea += 1

print ("*GetDataDone")
#form flag
flagstr = ""
for i in range (0, flaglen):
	tmp = 0;
	j = 7;
	while j >= 0: 
		tmp = tmp * 2 + flagarray[i][j]
		j -= 1;

	if tmp == 0:
		break

	flagstr += chr(tmp)

print (flagstr)
```

## 3.3 总结

本题关键在于指令流的获取，其中用到了jmp和call的地址计算。

- jmp/call xxx = 当前地址 + yyy + 指令长度 (其中xxx为无符号数，yyy为xxx的有符号版本；jmp的指令长度为2，call的指令长度为5)

## 3.4 PS

这个题是比赛后很久才做出来的。当时有想过使用idapython自行解析机器码来获取数据，但是不运行程序的话难以获得ret指令的跳转地址，所以放弃了。学习过SimpleVM后，今天尝试使用pintools的inscount来解决这个题，但是因为check不是按顺序来的，也不知道input长度，而且check时匹配的是位，难以设置输入参数，所以也没有成功。

向大佬请教后，了解到idapython确实可以解这个题。于是我再次想到了之前的方法，同样的再次卡在ret那里，但意外发现check中ret相当于nop，这样才“愉快”的获得了flag。*（十分愉快，毕竟机器码的解析踩了一下午的坑）*

大佬讲也可以使用pintools或angr来获取指令流，先挖个坑，可能之后学习pin和angr的时候会来填。