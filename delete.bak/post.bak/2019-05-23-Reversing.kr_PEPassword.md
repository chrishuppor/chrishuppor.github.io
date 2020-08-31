# 15 PEPassword

一个被输入框欺骗的题，一个很容易不知所云的题。

## 15.1 解题过程

1. 解压后没有看到readme，只有两个exe，一个叫original，一个叫packed。

   1. 运行original程序，弹出一个提示框，其中有”Password is“的字符，但是后面却是不可见字符。

      ![图1 original程序界面](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-06-36.PNG)

   2. 运行packed程序，弹出一个输入框，提示输入password，可以随意输入字符，但却找不到check按钮。

      ![图2 packed程序界面](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-07-39.PNG)

   3. 分析二者关系，从名称上看packed程序似乎是original程序加壳得到的。结合二者运行情况，可以猜测作者目的是用户在packed程序中输入一个正确的字符串，然后packed程序自解压成类似original的程序，弹出一个带有正确password的提示框。

   4. 拖入PEiD查看，果然original没壳，packed添加了PE password的壳。

2. 既然check在packed程序中，那么packed程序就是分析的重点。但在此之前我还是用OD查看了一下original程序。这个是一个十分简单的程序，除了提示字符串，程序中硬编码了一些字符，这些字符经过一轮异或操作后被MessageBox显示出来，只不过经过操作后这些字符都变成了不可见字符。因此，可以想到packed程序正确解密后会将这些字符修改成一些指定字符，经过变换后变为flag。

3. packed程序是加壳的程序，难以使用IDA分析，直接拖入OD。

   1. 查看packed程序使用的文本框字符获取函数

      之前使用PEView查看packed的IAT，只发现了LoadLibrary和GetProcAddress，说明这个壳会“手工”获取API地址。因此在GetProcAddress函数下断点就可以知道这个程序用到了哪些函数。

      结果没有发现文本框字符串获取函数。与字符获取有关的函数仅有GetMessageA。

   2. 看来Packed是使用GetMessageA逐个字符进行check。因此需要对GetMessageA下断点。注意，这里不可以用普通断点，因为GetMessageA是接收所有消息的，包括鼠标移动、定时器等，因此必须使用条件断点，在其获得键盘消息时中断。中断条件是[[ESP + 4] + 4] == 0x101.

   3. 程序会在用户输入字符时中断。此时查看堆栈可以找到返回地址0x409092.

      ![图3 找返回地址](https://chrishuppor.github.io/image/Snipaste_2019-05-23_22-28-03.PNG)

   4. 运行至0x409092，单步运行程序，会发现之后调用了TranslateMessage和DispatchMessage。在0x4090AB处有一个比较，如果没有通过则会跳转回GetMessageA之前，所以这里应该是需要通过的。

      1. 既然是比较，如果不是硬编码的就一定有修改被比较值的地方，因此将程序拖入IDA，查看[EBP+0x402A3E]的引用。如下，发现仅在0x4091AB的位置对[EBP+0x402A3E]进行了inc操作。

         ![图4](https://chrishuppor.github.io/image/Snipaste_2019-05-23_22-35-48.PNG)

      2. 在0x4091AB下断点，重新运行程序，输入字符，程序没有断下来。查看0x4091AB之前的代码，发现了一个jnz，所以可能是匹配失败所以跳走了。

      3. 在0x4091A6下断点，重新运行程序，输入字符，程序中断在0x4091A6。查看0x4091A6之前的代码发现了一个cmp。这个cmp之前有一个函数0x4091D8，猜测cmp中的EAX来自这个函数0x4091D8。

      4. 在IDA中查看sub_4091D8。模拟运行了一下，发现是一个hash函数，比较难求逆，所以这个函数应该不是突破口。从而这个路线应该都不是突破口。

   5. 爆破了0x4090AB之后，会发现之后有一个0x409200函数。跟进这个函数，如下，发现这个函数对0x401000起始的长度为0xfff字节的数据进行了修改。

      ![图5 sub_0x409200](https://chrishuppor.github.io/image/Snipaste_2019-05-23_22-58-47.PNG)

      其中，EBX是第一次经sub_4091DA计算得到的数据，EAX是第二次经sub_4091DA计算得到的数据，两次计算都与输入有关。因此猜测这里就是变相的check——如果当输入正确时，就可以将0x401000的代码解密正确，程序就可以弹出有正确pw的提示框；如果输入错误，程序就会跑飞。

      但是之前说过sub_4091DA函数为hash函数，难以求逆，看来需要其他手段使程序解密正确。

      进一步分析解密循环的寄存器的值发现，[EDI]的数据是一定的，ECX与EDX初始值确定为0，只有EAX和EBX的值是随输入变化的，因此如果能够知道EAX和EBX的初始值就能正确解密了。

      显然EAX是好求的。因为[EDI]解密结果由[EDI]原数据与EAX异或而来，已知解密结果应与original程序0x401000处代码一致，因此将packed程序0x401000起始的数据与original程序0x401000起始的数据异或就可以得到每轮循环中EAX的值。(注意内存数据存储使用小端序)

      EBX可以由EAX求得。解密循环开始前，EBX的初始值由硬编码的整数和输入经sub_4091DA函数计算而来，其中输入是未知的，所以这条路走不通。但是在解密循环中，如下，EAX第二个值由EAX初始值和EBX初始值计算而得，而且已知EAX的各个值，所以可以从这条路找到EBX初始值。

      ![图6 解密循环](https://chrishuppor.github.io/image/Snipaste_2019-05-24_16-46-54.PNG)

      将上图红框中的汇编代码整理的EAX2 = ROR4(EAX1 ^ROL4(EBX,  BYTE0(EAX1)), BYTE1(ROL4(EBX,  BYTE0(EAX1))))，其中EAX1表示计算前EAX值，EAX2表示计算后EAX值，ROR4(a,b)表示将32位的a循环右移b位，BYTE1(a)表示从低位开始取a的第二个字节，BYTE0(a)表示从低位开始取a的第一个字节。

      根据之前分析可知EAX1 =  0x014cec81 ^ 0xB6E62E17， EAX2 = 0x57560000 ^ 0x0d0c7e05，带入计算公式可得0x5a5a7e05 = ROR4(0xb7aac296 ^ROL4(EBX,  0x96), BYTE1(ROL4(EBX,  0x96)))。

   6. 求解ebx

      1. 令tmp = ROL4(EBX,  0x96)，则原公式简化为0x5a5a7e05 = ROR4(0xb7aac296 ^tmp, BYTE1(tmp))

      2. 找不到求解的办法，看来只能爆破了。如果从tmp入手，有0x100000000中可能，如果从BYTE1(tmp)入手有0x100中可能，因此从BYTE1(tmp)入手爆破。如果求得的tmp满足BYTE1(tmp)与假设的BYTE1(tmp)一致，则说明该tmp是正确的。

      3. 将0xb7aac296 ^tmp看做一个整体，令mid1 = 0xb7aac296 ^tmp，mid2 = BYTE1(tmp)则

         ```c
         EBX = ROR4(tmp, 0x96)
         tmp = 0xb7aac296 ^ mid
         mid1 = ROL4(0x5a5a7e05, mid2)
         ```

      4. 爆破脚本

         ```python
         a11 = 0x014cec81
         a21 = 0xB6E62E17
         
         a12 = 0x57560000
         a22 = 0x0d0c7e05
         
         #初始eax和第二个eax
         a1 = (a11 ^ a21)
         a2 = (a12 ^ a22)
         
         for mid2 in range(0, 0x100):
             count = mid2 % 32
             mid1 = (((a2 << count) | a2 >> (32 - count))) & 0xffffffff #注意这里，需要自己进行位数限制
             tmp = a1 ^ mid1
             if mid2 == ((tmp &0xff00)>>8):
                 count = 0x96 % 32
                 ebx = ((tmp >> count) | (tmp <<(32 - count))) & 0xffffffff
                 print (hex(ebx))
         ```

      5. 一共算出两个ebx，分别为0xa1beee22和0xc263a2cb。经测试，第二个ebx是正确的。

         ![图7 flagy](https://chrishuppor.github.io/image/Snipaste_2019-05-24_17-31-51.PNG)

      6. 最后对比了一下解密后的paecked和original的0x401000函数，如下，果然就是flag部分的字符不一样。

         ![图8 sub_401000对比](https://chrishuppor.github.io/image/Snipaste_2019-05-24_18-56-48.PNG)

## 15.2 小结

* 条件断点——条件表达式如何确定？

  * 断点条件一般是寄存器的值、函数参数的值等，在不确定的时候可以通过条件日志来确定。如图，选择从不中断程序、总是记录表达式的值。如果是函数入口，还可以选择总是记录函数参数。

    ![图9 条件日志配置](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-50-57.PNG)

  * 下完断点后，要清空原有日志。运行程序，查看日志中的数据，根据数据情况配置断点条件。例如本题中，在寻找GetMessageA参数中表示键盘消息的条件时，就用了这种方法。

    1. 总是记录函数参数，从不中断程序。运行程序后，分析其日志数据。

       ![图10 日志记录](https://chrishuppor.github.io/image/Snipaste_2019-05-23_21-57-18.PNG)

    2. 可以看到，函数第一个参数是一个指向MSG结构的指针，MSG结构中的Msg是WM_KEYUP时表示消息类型为按键。经查询，WM_KEYUP的值为0x101。也就是说中断条件是pMsg->Msg == 0x101。

    3. 从堆栈中可以看出，pMsg地址是ESP + 4，从MSG结构中可以看出Msg成员在MSG中偏移为0x4，所以pMsg->Msg可以表示为[[ESP + 4] + 4]。因此断点条件为[[ESP + 4] + 4] == 0x101。

  * 如果对断点条件表达式不是特别有把握，可以通过设置表达式来验证，比如将[[ESP + 4] + 4]设置为表达式，选择总是记录表达式的值。

* 本题关键有两点

  * 搞清楚破解的目的是让packed程序正确自解密成original的形式，但是字符串的值与original不同。
  * 突破定向思维，不与输入字符串纠缠，直接修改解密循环相关寄存器的初始值。

* 本题难点在于ebx爆破的方法

  * 查看他人的writeup时，发现除了爆破也没有其他的方法了。一开始我也直接从ebx开始爆破，但0x100000000的遍历似乎不太能接受，于是想到了从BYTE1(tmp)入手，计算复杂度陡降。