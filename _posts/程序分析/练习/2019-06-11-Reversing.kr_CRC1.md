---
layout: post
title: "Reversing.kr_CRC1"
pubtime: 2019-06-11
updatetime: 2019-06-11
categories: Reverse
tags: WriteUp
---

Reversing.kr的CRC1，难点在于能否想到中间相遇攻击。*（是剩下的四个题中最好捏的了，其余的捏的手都疼了）*


# 解题过程

1. 运行一下程序。

   需要用户输入数据，然后程序显示用户输入的数据WRONG还是CORRECT。

2. 将程序拖入IDA，在strings中找到WRONG，然后转到WRONG的调用。WRONG在函数sub_401070中被调用。

3. sub_401070函数较为简单，从头看起。

   1. 首先使用GetDlgItemTextA获取了一个字符串，要求这个字符串大小为8 Bytes。

      ```c
      GetDlgItemTextA(hDlg, 1000, &input0, 20) != 8
      ```

      使用RH查看就可以知道，ID为1000的控件就是那个文字输入框。

      ![图1 资源视图](https://chrishuppor.github.io/image/Snipaste_2019-06-11_14-19-51.PNG)

      由此可知，正确输入一共有8个字符。

   2. 接下来是一个略带蛇皮走位的赋值操作。

      ```c
      len_init = strlen(aHelloWelcomeTo) + 1;
      len = len_init;
      len_init >>= 2;
      qmemcpy(&szChar, aHelloWelcomeTo, 4 * len_init);
      ptr_aHelloWelcomeTo = &aHelloWelcomeTo[4 * len_init];
      ptr_szChar = &szChar + 4 * len_init66;
      v_2 = len & 3;
      v6 = 0;
      qmemcpy(ptr_szChar, ptr_aHelloWelcomeTo, v_2);
      v7 = &szChar;
      ```

      翻译过来就是strncpy(&szChar, aHelloWelcomeTo, strlen(aHelloWelcomeTo)); v7 = &szChar。

      简化一下就是strncpy(v7, aHelloWelcomeTo, strlen(aHelloWelcomeTo));

   3. 然后就是将用户输入引入v7数据。

      ```c
      do
      {
          v8 = *(&input0 + v6++);
          *v7 = v8;
          v7 += 16;
      }
      while ( v6 < 8 );
      ```

      这里将v7看做8*16的矩阵，将v7[i\][0]赋值为input[i]

   4. 然后对v7中数据进行了计算。

      ```c
      do
      {
          v11 = (unsigned __int8)v9 ^ (unsigned __int8)*(&szChar + v10);
          v12 = v9 >> 8;
          LODWORD(v9) = v12 ^ dword_4085E8[2 * v11];
          v13 = HIDWORD(v9) ^ dword_4085EC[2 * v11];
          ++v10;
          HIDWORD(v9) = v13;
      }
      while ( v10 < 256 );
      ```

      其中&szChar就是v7，并且根据v10 < 256可知v7的大小为256字节。根据后面的代码可知，最后得出的v9和v13就是匹配成功的关键。dword_4085EC[2 * v11]就是dword_4085E8[2 * v11 + 1]。

      查看dword_4085E8引用，发现在sub_401000函数中对其进行了赋值。很容易发现sub_401000函数其实就是crc_table的初始化函数，因此dword_4085E8数组就是crc_table的存储数组。

      另外，在sub_401000中还有对aHelloWelcome数据的修改。*（真是的，蛇皮走位的赋值还不行，还有犄角旮旯的数据修改，这就不怎么磊落了）*

   5. 接下来就是匹配的关键了：if ( (_DWORD)v9 != 0x5F695F6C || v13 != 0x676F5F67 )。当且仅当v9 == 0x5F695F6C  && v13 == 0x676F5F67 时程序才会提示出CORRECT。

      ```c
      do
      {
          v11 = (unsigned __int8)v9 ^ (unsigned __int8)*(&szChar + v10);
          LODWORD(v9) = (v9 >> 8) ^ dword_4085E8[2 * v11];
          HIDWORD(v9) = HIDWORD(v9) ^ dword_4085E8[2 * v11];
          ++v10;
      }
      while ( v10 < 256 );
      ```

4. 破解

   1. 使用IDA动态加载程序，使用IDAPython从内存中获取dword_4085E8数组和szChar原始数据。

      emmm...IDA崩了，所以自己手动算吧。

   2. 根据CRC原理设计中间相遇攻击，降低爆破空间，进而计算出flag。*(CRC原理简介可参考crc学习笔记)*

      脚本如下:

      ```python
      def initCRCtable(dividend):
      	crc_table = []
      	index_table = []
      	for i in range(0, 256):
      		index_table.append(0)
      
      	for i in range(0, 256):
      		crc = i
      		for j in range(0, 8):
      			tmp = crc & 0x1
      			crc >>= 1
      			if tmp == 1:
      				crc ^= dividend
      		crc_table.append(crc) #for improve crc_calc rapid
      		index_table[crc >> 56] = i #to find the value of crc.table[index] with the result of (lastcrc ^ crc.table[index]), so we can know lastcrc, then we can uncalc_crc
      
      	return crc_table, index_table
      
      
      def calcCRC(inputstr, crc_table):
      	iLen = len(inputstr)
      	crc = 0
      	for i in range(0, iLen):
      		index = (inputstr[i] ^ crc) & 0xff
      		crc >>= 8
      		crc ^= crc_table[index]
      	return crc
      
      
      def unCalcCRC(inputstr, rescrc, crc_table, index_table):
      	finalcrc = rescrc
      	iLen = len(inputstr)
      	i = iLen - 1
      	while(i >= 0):
      		index = index_table[finalcrc >> 56]
      		finalcrc ^= crc_table[index]
      		finalcrc <<= 8
      		finalcrc |= index ^ inputstr[i]
      		i -= 1
      	return finalcrc
      
      
      def calcFlagFormer(welcome_array,crc_table, tar_crc):
      	calcFlagFormerArray = []
      	for i in range(32, 127):
      		print ("[+]in former ", i - 31)
      		for j in range(32, 127):
      			for m in range(32, 127):
      				for n in range(32, 127):
      					welcome_array[0x00] = i;
      					welcome_array[0x10] = j;
      					welcome_array[0x20] = m;
      					welcome_array[0x30] = n;
      					crc = calcCRC(welcome_array, crc_table)
      					if crc == tar_crc:
      						flagarray.append(i)
      						flagarray.append(j)
      						flagarray.append(m)
      						flagarray.append(n)
      						return calcFlagFormerArray
      					if tar_crc == 0:
      						calcFlagFormerArray.append(crc)
      
      	return calcFlagFormerArray
      
      
      def calcFlagLatter(welcome_array, rescrc, crc_table, index_table, tar_crc):
      	calcFlagLatterArray = []
      	for i in range(32, 127):
      		print ("[+]in latter ", i - 31)
      		for j in range(32, 127):
      			for m in range(32, 127):
      				for n in range(32, 127):
      					welcome_array[0x00] = i;
      					welcome_array[0x10] = j;
      					welcome_array[0x20] = m;
      					welcome_array[0x30] = n;
      					crc = unCalcCRC(welcome_array, rescrc, crc_table, index_table)
      					if crc == tar_crc:
      						flagarray.append(i)
      						flagarray.append(j)
      						flagarray.append(m)
      						flagarray.append(n)
      						return calcFlagLatterArray
      					if tar_crc == 0:
      						calcFlagLatterArray.append(crc)
      						
      	return calcFlagLatterArray
      
      def writeToFile(a, filepath):
      	fw = open(filepath, "w")
      	for item in a:
      		fw.write(str(item)+"\n")
      	fw.close()
      
      dividend = 0xC96C5795D7870F42
      (crc_table, index_table) = initCRCtable(dividend)
      #-----------------
      
      tmpstr = '_[Hello___Welcome To Reversing.Kr]__The idea of the algorithm came out of the codeengn challenge__This algorithm very FXCK__But you can solve it!!__Impossible is Impossible_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_[]_()_['
      welcome_array = []
      for i in tmpstr:
      	welcome_array.append(ord(i))
      welcome_array.append(0) #here is a WTF——for 1h time wasted
      
      tmpReplace = 0x2E9013A335DE51E7;
      for i in range(208, 216):
      	tmp = tmpReplace & 0xff
      	welcome_array[i] = tmp
      	tmpReplace >>= 8
      '''
      a1 = calcFlagFormer(welcome_array[0:64],crc_table, 0)
      print ("[+]calcFlagFormer over.")
      a1.sort()
      print ("[+]FormerCrc sort over.")
      writeToFile(a1, "c:\\users\\chris\\desktop\\mycrc1.txt")
      print ("[+]FormerCrc write over.")
      del a1[:]
      '''
      
      rescrc = 0x676F5F675F695F6C
      rescrc = unCalcCRC(welcome_array[128:], rescrc, crc_table, index_table)
      a2 = calcFlagLatter(welcome_array[64:128], rescrc, crc_table, index_table, 0)
      print ("[+]calcFlagLatter over.")
      a2.sort()
      print ("[+]LatterCrc sort over.")
      
      fr = open("c:\\users\\chris\\desktop\\mycrc1.txt", "r")
      
      tmpa1 = fr.readline()
      a2count = 0
      while(len(tmpa1) > 2):
      	tmp = int(tmpa1[:len(tmpa1) - 1])
      	while (tmp > a2[a2count]):
      		a2count += 1
      	while (tmp < a2[a2count]):
      		tmpa1 = fr.readline()
      		tmp = int(tmpa1[:len(tmpa1) - 1])
      	if tmp == a2[a2count]:
      		break
      
      tar_crc = a2[a2count]
      del a2[:]
      
      flagarray = []
      calcFlagFormer(welcome_array[0:64],crc_table, tar_crc)
      calcFlagLatter(welcome_array[64:128], rescrc, crc_table, index_table, tar_crc)
      
      flagstr = ''
      for i in flagarray:
      	flagstr += chr(i)
      print (flagstr)
      ```

   # 小结

   * 很纯粹的CRC的应用，如果会CRC的话会感觉十分容易。*（如果不会，那怨谁，谁教你不会呢）*

   # 后记

   一道crpyto的逆向题，应用到了密码学和编码学的知识，没有其他的弯弯绕，就是刚算法的硬题了。

   这个题拖了很久，一个是最近忙着打KCTF，一个是需要恶补CRC，一个是程序调了一段时间才跑出正确数据，一个是其他事情也比较多......尤其是历史遗留问题，尽管有心理准备，但打击还是不小。尽管少年时觉得我命由我不由天，尽管现在也因为这句话在拼搏，但是有时候老天要整我，我能有什么办法！命运岂有道理？谢逊说的对。
