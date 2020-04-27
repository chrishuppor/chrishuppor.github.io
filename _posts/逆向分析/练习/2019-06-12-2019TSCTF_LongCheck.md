---
layout: post
title: "2019TSCTF_LongCheck"
pubtime: 2019-6-12 22:10:6
updatetime: 2019-6-12 22:10:6
categories: Reverse
tags: WriteUp
---

2019年TSCTF的一道Reverse，主要考察指令流获取。[[题目及IDB文件下载](https://github.com/chrishuppor/attachToBlog/tree/master)]


# 解题思路

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

   * je <x\> = 跳转到（当前地址 + x +2)，je指令为0x74

   * jmp <x\> = 跳转到（当前地址 + x +2)，jmp指令为0xEB

   在运行脚本时发现还有其他地方有0x74和0xEB，需要忽略这些地方：

   * call sub_offset2b0: 有时0x74和0xEB会出现在call指令中，幸运的是check函数中仅有call sub_offset2b0指令出现了0x74和0xEB。因此可以在当发现机器码E8，计算其跳转的地址，如果为offset2b0则步过该指令。
   * mov ebx, 5eb: 该指令机器码为66 bb eb 05，因此当发现eb时，如果之前两个字节为66 bb，则应该步过。但是有时会直接jmp到eb所在地址，此时需要解析为jmp函数。

   check函数中有很多ret指令，本来无法自行解析，但本函数ret指令相当于nop。另外需要注意的是：shr al, 1的指令有其特殊的机器码。

5. 根据以上分析编写脚本获取flag。脚本运行结果如下：

![图1 脚本运行结果](https://chrishuppor.github.io/image/Snipaste_2019-06-12_21-38-43.PNG)

# 解题脚本

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

# 总结

本题关键在于指令流的获取，其中用到了jmp和call的地址计算。

* jmp/call xxx = 当前地址 + yyy + 指令长度 (其中xxx为无符号数，yyy为xxx的有符号版本；jmp的指令长度为2，call的指令长度为5)

# PS

这个题是比赛后很久才做出来的。当时有想过使用idapython自行解析机器码来获取数据，但是不运行程序的话难以获得ret指令的跳转地址，所以放弃了。学习过SimpleVM后，今天尝试使用pintools的inscount来解决这个题，但是因为check不是按顺序来的，也不知道input长度，而且check时匹配的是位，难以设置输入参数，所以也没有成功。

向大佬请教后，了解到idapython确实可以解这个题。于是我再次想到了之前的方法，同样的再次卡在ret那里，但意外发现check中ret相当于nop，这样才“愉快”的获得了flag。*（十分愉快，毕竟机器码的解析踩了一下午的坑）*

大佬讲也可以使用pintools或angr来获取指令流，先挖个坑，可能之后学习pin和angr的时候会来填。