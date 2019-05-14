微简RE challenge网站，my_4ear_3hr1s 就系我啦，第9题解题过程及题后思考记录如下。

# Reversing.kr_Ransomware

## 解题过程

1. 查看文件

   有一个exe，一个file，一个readme.

   * 查看readme，内容如下。

     只知道要解密file，又提到exe，难道是说这个file可能跟exe有关?猜测Flag可能是解密密钥。

     ```
     Decrypt File (EXE)
     By Pyutic
     ```

   * 使用UE查看file，一个二进制文件。

2. 把run.exe拖进PEiD，有一个upx壳。

   使用upx脱壳。

3. 把run.exe拖进IDA

   1. 加载了很久——这个很奇怪，这个文件又不大*（后面知道了这个程序有很多垃圾指令，不知道是不是这个原因）*

   2. 查看string

      string很少，但还是有一个关键的字符串“key:”。

      ![Snipaste_2019-05-14_11-06-25.PNG](Snipaste_2019-05-14_11-06-25.PNG)

   3. 转到key的引用，按F5，但是因为函数太大而反编译失败。

      查看附近的汇编代码，发现key引用之前有大片的无用代码，如下。

      ```
      .text:0044A763                 push    eax
      .text:0044A764                 pop     eax
      .text:0044A765                 push    ebx
      .text:0044A766                 pop     ebx
      .text:0044A767                 pusha
      .text:0044A768                 popa
      .text:0044A769                 nop
      .text:0044A76A                 push    eax
      .text:0044A76B                 pop     eax
      .text:0044A76C                 push    ebx
      .text:0044A76D                 pop     ebx
      .text:0044A76E                 pusha
      .text:0044A76F                 popa
      .text:0044A770                 nop
      .text:0044A771                 push    eax
      .text:0044A772                 pop     eax
      ```

      说明程序除了加壳，还做了混淆，使函数变得巨大，从而使IDA反编译失败。

   4. 去掉混淆

      垃圾代码不影响程序逻辑，可以直接去掉。去掉的方法是将本函数中第一条垃圾指令前的代码复制到垃圾指令后第一条有意义代码的前面。

      本题比较简单，直接从函数入口开始到key的引用都是垃圾指令，因此可以直接将函数入口代码复制到key引用之前。*（ps:当前地址属于哪个函数可以从窗口下方看到）*

      也就是将函数开头的0x40135e0到0x4135e7的二进制代码复制到0x44a76c到0x44a774。

      之后，"右键->edit function"修改main函数入口地址为0x44a76c，然后F5就可以成功了。

   5. 查看反编译后的main函数

      1. 其中有一个sub_401000函数，查看这个函数，也是一堆垃圾指令，看来也是混淆过的。使用idapython查找其有意义的指令位置，结果发现整个函数就没有用。脚本如下：

         ```python
         import idaapi
         
         ea = 0x401006
         command = ["push", "pop", "pusha", "popa", "nop"]
         while idc.GetMnem(ea) in command:
         	ea = idc.NextHead(ea)
         print(hex(ea)) #最终的输出是sub_401000的retn地址
         ```

         因此，可以将程序中所有的call sub_401000去掉。脚本如下：

         ```python
         import idaapi,ida_bytes
         
         ea = 0x44A770
         ed = 0x44a983
         x = 0x90
         while ea < ed:
             if idc.GetDisasm(ea) == 'call    sub_401000':
                 for i in range(0, 5):
                     ida_bytes.put_bytes(ea + i, chr(x))
             ea = idc.NextHead(ea)
         ```

      2. 

4. 

## 小结

* 刚开始看readme时会一头雾水，因为把Decrypt File理解成了“解密某个文件”，而实际上这里的File是文件中的file的文件名。*（如果是解密某个文件，应该为Decrypt a File。英语还得学呀，要不然注意不到这些细节的差别。）*
* 代码混淆——垃圾代码