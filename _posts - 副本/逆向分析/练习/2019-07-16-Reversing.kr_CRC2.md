---
layout: post
title: "Reversing.kr_CRC2"
date: 2019-07-16 8:8:8
categories: Reverse
tags: WriteUp
---

Reversing.kr的CRC2，使用到了CRC算法的线性特征以及掩码的思想。


# 解题过程

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

# 小结

解题关键在于CRC算法线性特征的利用，引入了掩码的思想，还考察了线性方程求解的知识。

在编写sage代码时需要注意与python的区别：

* ^在python中表示异或，在sage中表示乘方。sage中使用^^表示异或。([sage官方文档](http://doc.sagemath.org/html/en/faq/faq-usage.html))
* 二维数组python中下标使用[i\][j]，sage中使用[i, j]

# 感想

第一次切实意识到线性代数的应用意义，也是第一次使用sage，实在强大。

# 大佬WP

[https://github.com/DoubleLabyrinth/reversing.kr/tree/master/CRC2](https://github.com/DoubleLabyrinth/reversing.kr/tree/master/CRC2)