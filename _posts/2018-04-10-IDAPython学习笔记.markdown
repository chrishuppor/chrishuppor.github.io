---
layout: post
title: "IDApython学习笔记"
date: 2018-04-10 8:8:8
categories: LearningNote
tags: IDAPython
---
&#160;&#160;&#160;&#160;&#160;&#160;&#160;IDApython，对使用中学习的函数整理记录如下。曾使用IDApython辅助程序分析，还是很好用的。

# 普通地址操作
1.  MinEA():获取本模块最小地址  
    MaxEA():获取本模块最大地址  
    eg：
    ```
    ea = MinEA()
    print hex(ea)
    output: 401000 #idapython中不会显示数字是多少进制，如果不用hex()，默认为十进制
    ```
2. ea = here():获取当前光标所在地址  
   ea = idc.ScreenEA():获取当前光标所在地址
3. 地址加减
    * 地址直接加减：ea = ea + 1
    * 函数
        * idc.NextAddr(ea)地址步进1
        * idc.PrevAddr(ea)地址退后1
4. 判断地址是否有效——    idaapi.BADADDR  
    eg:
    ```
    ea = here()
    if not (ea == idaapi.BADADDR):
        print "valid addr"
    ```

# 普通字符串
&#160;&#160;&#160;&#160;&#160;&#160;&#160;Names() #获取程序中所有符号的列表（和字符串不是一个东西，但是字符的信息可以在这里找到，而且对标点进行了变化）  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;Strings() #获取字符串列表  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;eg：
```
for item in Strings():
    print("%s"%item) #problem:"\n"不会显示出来，而是直接作为了回车
    print Strings.StringItem.__str__(item) #这个也行
```

# 二进制数据操作

1. dump内存数据
```python
import idaapi
data = idaapi.dbg_read_memory(start_address, data_length)
# data就是内存数据了，不过是将十六进制的数据按字符的格式来存储的，比如'\x00'
```
2. 以整数的形式获取指定内存数据
```python
Byte(<地址>) #获取的结果是整数，例如：0xff
```

3. 将指定地址开始的数据修改为x
```python
import ida_bytes
ida_bytes.put_bytes(ea, x) #其中type(x) == str
```

# 指令

1. 指令地址
    * idc.NextHead(ea)返回下一条指令地址
    * idc.PrevHead(ea)返回上一条指令地址
2. 指令字符串
    * idc.GetDisasm(ea) #获得反汇编的字符串 output eg:push 0
    * idc.GetMnem(ea) #获取助记符或者指令名称 output eg:push
    * idautils.FuncItems(ea) #获取该函数的所有汇编指令地址列表
    * GetInstructionList() #获取汇编指令表(一堆汇编指令助记符，没有操作数的)
3. 操作数
    * idc.GetOpnd(ea，long n)：获取操作数的助记符，第一个参数是地址，第二个long n是操作数索引（第一个操作数索引是0和第二个是1）。
    * idc.GetOpType(ea, n)：返回操作数类型（**PS：如果该指令有n个操作数，则操作数的index为0到n-1,GetOpType(ea, n)返回0**）
        * o_void：不含有操作数的指令返回0，如：retn
        * o_reg：寄存器，如：pop edi
        * o_mem：内存引用，如：cmp ds:dword_A152B8, 0
        * o_phrase：寄存器寻址，如：mov [edi+ecx], eax
        * o_displ：偏移寻址，如：mov eax, [edi+18h]
        * o_imm：直接数，如：add esp, 0Ch
        * o_far：x86和x86_64很少用到
        * o_near：x86和x86_64很少用到
    * idc.OpOff(ea, n, base)：将操作数转换为offset，base为offset基地址，n为index
    * idc.GetOperandValue(ea, n)：得到操作数的值，eg:对于push eax，可以返回eax的值

## eg:打印带有立即数的指令（格式：指令_立即数）
```
ea = MinEA() #exe起始地址
ea_end = MaxEA() #终止地址

print_flag = 0; #指令是否有立即数，有为1，无为0

while ea <= ea_end: #从起始地址遍历到终止地址
    #获得反汇编的字符串 eg:push 0
    disassembled_str = idc.GetDisasm(ea)
    disassembled_name = idc.GetMnem(ea) #指令名称（如果指令是db、dd之类的非汇编指令，返回""）
        
    if ("" == len(disassembled_name):
        ea = idc.NextHead(ea)
        continue
    
    #获得操作数个数（通过‘,’个数:有n个逗号就有n+1个操作数）
    operands_sum = disassembled_str.count(',', 0, len(disassembled_str)) + 1
    
    print_str = disassembled_name #初始化输出字符串
    #获取操作数中的立即数
    for i in xrange(1, operands_sum):
        opnd = idc.GetOpnd(ea, i)
        if (idc.GetOpType(ea, i) == o_imm):#如果有立即数，就构造输出字符串
            print_flag = 1
            print_str = print_str + '_' + (opnd)
    
    if print_flag == 1:#如果有立即数就输出
        print print_str    
        
    print_flag = 0
    
    # 取下一条指令地址
    ea = idc.NextHead(ea)
```

# PE Header信息获取

## 程序类型
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idaapi.get_file_type_name() #返回程序类型字符串，output eg:Portable executable for 80386 (PE)

## 基址
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idaapi.get_imagebase()

## 段信息
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idautils.Segments() #return List of segment start addresses  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idc.SegName(seg) #返回段名称,seg是段中任意地址  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idc.SegStart(seg) #返回段起始地址  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idc.SegEnd(seg) #返回段结束地址  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idc.NextSeg(ea) #返回下一个段起始地址，ea可以是当前段中任意地址  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idc.SegByName(name) #根据名称返回段起始地址

### eg:获取段名
```
import idautils
for seg in idautils.Segments():
    print idc.SegName(seg)
```

## 导入表
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idaapi.get_import_module_qty() #获取import的模块个数  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idaapi.get_import_module_name(i) #获取第i个import模块的名称  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;idaapi.enum_import_names(i, imp_cb) #列举第i个inport模块的导入函数名称，imp_cb是回调函数参数是(ea, name, ord)，返回值是TRUE/FALSE(参数和返回值是固定的)

### eg:打印导入表信息
```
import idaapi

def imp_cb(ea, name, ord):
    if not name:
        print "%08x: ord#%d" % (ea, ord)
    else:
        print "%08x: %s (ord#%d)" % (ea, name, ord)
    # True -> Continue enumeration
    # False -> Stop enumeration
    return True #不return TRUE的话，会默认是False，停止遍历
#------------------------------------------------------------------------

nimps = idaapi.get_import_module_qty()

for i in xrange(0, nimps):
    name = idaapi.get_import_module_name(i) #获取第i个模块名称
    if not name:
        print "Failed to get import module name for #%d" % i
        continue

    print "Dll Name-> %s" % name
    idaapi.enum_import_names(i, imp_cb) #获取导入函数
```

# 函数信息
1. 地址
    * idautils.Functions() #返回所有函数地址的列表
    * idc.NextFunction(ea) #返回下一个函数起始地址
    * idc.PrevFunction(ea) #返回上一个函数起始地址 
2. idc.GetFunctionName(ea) #返回函数名称，ea为该函数内任意地址  
3. idaapi.get_func(ea) #返回fun_t类型，其中有诸如startEA, endEA属性用于返回函数起始和结束地址
4. idc.GetFunctionAttr(ea, attr) #返回当前函数属性
5. idc.GetFunctionFlags(func_a) #返回函数类型
    * FUNC_NORET- 函数无返回值
    * FUNC_FAR- 不常用
    * FUNC_USERFAR- 不常用
    * FUNC_LIB- 函数为标准库函数
    * FUNC_STATIC- 函数为static
    * FUNC_FRAME- 函数使用栈帧指针ebp，一般为push ebp; mov ebp, esp; sub esp, xxh
    * FUNC_BOTTOMBP- 函数使用栈指针，与FUNC_FRAME类似
    * FUNC_HIDDEN- 函数被View - Hide折叠
    * FUNC_THUNK

## eg:获取库函数
```
import idc, idautils

for func in idautils.Functions(MinEA(), MaxEA()):
    flags = idc.GetFunctionFlags(func)
    # Ignore library functons
    if (flags & FUNC_LIB) and (idc.GetFunctionName(func)):#表示函数属性为FUNC_LIB且有函数名
        print idc.GetFunctionName(func)
```

# 其他
&#160;&#160;&#160;&#160;&#160;&#160;&#160;GetInputFileMD5() #获取输入文件的md5

# 程序静态信息提取脚本

[程序静态信息提取脚本示例:static_extract412.py](https://github.com/chrishuppor/src/blob/master/IDAPython/static_extract412.py)

# 参考链接
1. [idautils.py函数功能总结by真·技术宅](https://www.0xaa55.com/thread-1586-1-1.html)
2. [IDAPython备忘byV191](http://blog.sina.com.cn/s/blog_9f5e368a0102wnmm.html):对于指令、操作数、函数、引用、查找、注释等常用的功能有全面的总结，是本文的主要参考
3. [IDAPython模块+函数+结构说明文档](https://www.hex-rays.com/products/ida/support/idapython_docs/index.html):有的查不到，仅做参考;使用函数时注意import对应的库
4. [Python idaapi.BADADDR() Examples](https://www.programcreek.com/python/example/84850/idaapi.BADADDR):英文资料，不明觉厉就列在这里充门面...