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

   求解 [b00; b01; b02; ... b07; b10; b11; ...; b12.7]。*（可以直接参考DoubleLabyrinth大佬的wp，比我讲的清楚多了）*




   获取byte_410F00初始化后的数据，脚本如下。需要注意的是，从401490中可以看出byte_410F00大小为256Bytes，但脚本得到的仅有222个字节，余下的0需要自行补全。

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






<https://github.com/DoubleLabyrinth/reversing.kr/tree/master/CRC2>