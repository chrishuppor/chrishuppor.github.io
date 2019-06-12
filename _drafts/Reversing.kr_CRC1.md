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

      ![图1 资源视图](Snipaste_2019-06-11_14-19-51.PNG)

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

      查看dword_4085E8引用，发现在sub_401000函数中对其进行了赋值。

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

