Reversing.kr的第十六题，是一个mac系统的程序，之前从未见过mac相关的逆向，可能是我经验不够，不过这个题目难度与EasyELF有一拼。

# Reversing.kr_HateIntel

## 解题步骤

1. 没有mac系统的机器，暂时不能动态调，先拖进IDA看看情况。

   1. 看上去很简单，函数很少，也能反编译。

   2. 查看字符串，有很明显的提示信息"Wrong"和"Correct"，查找其引用，发现了sub_2224函数。

   3. sub_2224

      代码如下，先接受了一个字符串输入，然后经过 calc_input_232C函数处理，再进入一个比较循环。如果比较通过就提示"Correct"，有一个失败就会提示"wrong"。这个循环显然是对input数据的check，因为&vars0 - 0x5C = &input_str，而且循环变量v3 = strlen(&input_str)，说不是对input数据的check都没有人信。查看byte_3004，发现都不是可见字符，说明之前对input进行了变换。恰巧在check之前有一个以input为参数的calc_input_232C函数。

      ```c
      int input_check_2224()
      {
        char input_str; // [sp+4h] [bp-5Ch]
        int v2_4; // [sp+54h] [bp-Ch]
        int v3; // [sp+58h] [bp-8h]
        int i; // [sp+5Ch] [bp-4h]
        char vars0; // [sp+60h] [bp+0h]
      
        v2_4 = 4;
        printf("Input key : ");
        scanf("%s", &input_str);
        v3 = strlen(&input_str);
        calc_input_232C((signed __int32)&input_str, v2_4);
        for ( i = 0; i < v3; ++i )
        {
          if ( *(&vars0 + i - 0x5C) != byte_3004[i] )
          {
            puts("Wrong Key! ");
            return 0;
          }
        }
        puts("Correct Key! ");
        return 0;
      }
      ```

   4. 查看calc_input_232C函数.

      核心逻辑如下，其中v2 = 4， v3 = &input。整个逻辑比较简单，只是内层循环跟正常人的写法不一样，翻译过来就是依次调用rol_2494函数处理输入的每个字符，整个字符串一共处理4次。

      那么处理的关键就是rol_2494函数了。

      ```c
      for ( i = 0; i < v2; ++i )
      {
          for ( j = 0; ; ++j )
          {
              input_str = strlen(v3);
              if ( input_str <= j )
                  break;
              v3[j] = rol_2494(v3[j], 1);
          }
      }
      ```

   5. 查看rol_2494函数

      又是一个具有迷惑性的一个函数，翻译过来就是将a1循环左移1位

      ```c
      int __fastcall ror_2494(unsigned __int8 a1, int a2_1)
      {
        int v3; // [sp+8h] [bp-8h]
        int i; // [sp+Ch] [bp-4h]
      
        v3 = a1;
        for ( i = 0; i < a2_1; ++i )
        {
          v3 *= 2; //左移一位
          if ( v3 & 0x100 ) //判断最高位是否为1
            v3 |= 1u; //如果最高位为1，则将末位置为1
        }
        return (unsigned __int8)v3;//返回8位的v3
      }
      ```

   6. 综上所述，calc_input_232C函数的作用就是依次将input中的字符循环左移4位，移位结果与byte_3004中数据比较。因此，flag就是依次将byte_3004中的数据循环右移4位。脚本如下：

      * byte_3004数据获取脚本

        ```python
        import idaapi
        ea = 0x3004
        
        res = []
        for i in range(0, 29):
            res.append((Byte(ea)))
            ea += 1
        
        print res
        ```

      * flag"翻译"脚本

        ```python
        a = [68, 246, 245, 87, 245, 198, 150, 182, 86, 245, 20, 37, 212, 245, 150, 230, 55, 71, 39, 87, 54, 71, 150, 3, 230, 243, 163, 146, 0]
        
        res = ''
        for item in a:
            tmp = (item >> 4) | ((item << 4)&0xff)
            res += chr(tmp)
        
        print (res)
        ```

## 小结

没什么好说的，只是想起了领袖的名言“一切反动派都是纸老虎”。无论题目怎么变化，本质是一样的。

* 要学会将不太正常的代码翻译成正常代码，这样就会简单许多。