# 8 EasyELF

emmm...这个题大概是按分类放的...

## 8.1 解题过程

1. 拖入IDA

   查看字符串，找到了wrong。查看wrong的引用，找到了main，十分简单。

   ```c
   int __cdecl main()
   {
     write(1, "Reversing.Kr Easy ELF\n\n", 0x17u);
     input_8048434();
     if ( sub_8048451() == 1 )
       sub_80484F7();
     else
       write(1, "Wrong\n", 6u);
     return 0;
   }
   ```

   查看input_8048434()，发现是一个scanf。

   查看sub_8048451()，发现了很多比较，如下：

   ```c
   _BOOL4 sub_8048451()
   {
     if ( byte_804A021 != '1' ) //input[1] == '1'
       return 0;
     inputstr_804A020 ^= 0x34u;
     byte_804A022 ^= 0x32u;
     byte_804A023 ^= 0x88u;
     if ( byte_804A024 != 'X' ) //input[4] == 'X'
       return 0;
     if ( byte_804A025 )//input[5] == '\0',说明字符串到此结束
       return 0;
     if ( byte_804A022 != 0x7C ) //input[2] ^ 0x32u == 0x7c
       return 0;
     if ( inputstr_804A020 == 0x78 )//input[0] ^ 0x34u == 0x78
       return byte_804A023 == 0xDDu;//input[3] ^ 0x88u == 0xdd
     return 0;
   }
   ```

   没有input字符串的偏移，而是直接使用了字符地址。

   不过也不难看出，因为inputstr_804A020是接收输入字符串的指针地址，那么接下的地址当然是存储input[1]、input[2]...的。

## 8.2 小结

* 看出byte_804A021、byte_804A022这些是input[1]，input[2]就没什么了...十分友好的一道题