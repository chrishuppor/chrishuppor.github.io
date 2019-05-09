---
layout: post
title: "Reversing.kr_Direct3DFPS"
date: 2019-5-8 14:58:4
categories: WriteUp
tags: Reversing_kr
---


微简RE challenge网站，my_4ear_3hr1s 就系我啦，第8题解题过程及题后思考记录如下。

# Reversing.kr_Direct3DFPS

## 解题过程

1. 照例拖进PEiD、virustotal、PEView。发现IAT中有MessageBox，可能用于显示Flag。没有其他异常。

2. 这是一个GUI程序，拖进RH，没有发现有用信息。

3. 运行程序

   打开程序的瞬间是想拒绝的，莫非是个Misc？走几步，开几枪，大概了解程序的功能了——杀掉所有怪物，弹出Flag。

   但肯定没这么直接，否则就是电竞题了。

4. 这个程序是一个完整的FPS游戏，规模较大，有大量的代码，所以肯定不能直接上OD，先拖进IDA看看。

   1. 查看Strings

      看到两个有意思的字符串：Game Over和Game Clear。这很可能是游戏结束和游戏胜利的提示。

      ![Snipaste_2019-05-08_22-28-20.PNG](..\image\Snipaste_2019-05-08_22-28-20.PNG)

   2. 追到Game Over的引用

      Game Over直接出现在WinMain函数中，让人感觉这个程序很是开门见山。

      如图，在一个if条件中显示出GameOver。联想到FPS中主角死亡时GameOver，也就是主角HP = 0时会GameOver，所以myBlood_407020一定是用于存储主角HP的。

      ![Snipaste_2019-05-08_22-56-18.PNG](..\image\Snipaste_2019-05-08_22-56-18.PNG)

      if ( myBlood_407020 <= 0 )之外的部分，也就是主角还活着的时候，应该是游戏正常运转所需要的各种操作，例如游戏画面更新、主角信息更新、怪物信息更新等的处理，并且在进行这些处理的过程中肯定要有游戏结束的判断，如果胜利则给出Flag。

   3. 追到GameClear的引用

      GameClear出现在一个很小的函数中，追踪这个函数的引用，发现它就出现在主角死亡判断后面，也就是上图中的	Game_clear_4039C0。

      代码如下，当满足一个条件时，会弹出一个MSGBOX。查看407028，这是一个长得很像Flag的字符串，但有些不是可见字符，肯定是加密过了，而解密肯定与游戏有关。

      其中， (signed int)result >= (signed int)&unk_40F8B4这个条件应该是在表示怪物全部死亡，循环判断*result != 1 应该表示怪物状态，如果怪物死亡就继续遍历，否则跳出循环。

      所以result从0x409194开始，每次+132，result = 0x40F8B4时结束——从第一只怪物的信息开始遍历，每个怪物信息大小为132，最后一只怪物信息存放于0x40F8B4。

      ```c++
      int *Game_clear_4039C0()
      {
        int *result; // eax
        result = GogoomaState_409194; //第一只怪物信息记录的地址
        while ( *result != 1 ) //遍历怪物信息，如果有怪物状态不是死亡就跳出循环
        {
          result += 132;//说明一个怪物信息struct大小为132
          if ( (signed int)result >= (signed int)&unk_40F8B4 )//说明unk_40F8B4应该是最后一只怪物信息地址，所以怪物信息存储在0x409194和0x40F8B4之间
          {
            MessageBoxA(hWnd, &FLAG_407028, "Game Clear!", 0x40u);
            return (int *)SendMessageA(hWnd, 2u, 0, 0);// Send Destroy
          }
        }
        return result; 
      }
      ```

   4. 寻找Flag的解密

      要解密Flag，肯定要引用加密后的字符串，也就是FLAG_407028。

      查看FLAG_407028的引用，如下，也是一个很小的函数。该函数对FLAG的修改十分明显：*((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4]。而且其他位置也没有跟多的FLAG的操作，所以这里就是要找的解密部分。

      ```c++
      int __thiscall ChangeFlag_403400(void *pVoid)
      {
        int result; // eax
        int HP_pos; // ecx
        int Gogoma_HP; // edx
      
        result = mayHitGogoomPos_403440(pVoid);
        if ( result != -1 )
        {
          HP_pos = 0x84 * result;
          Gogoma_HP = gogooma_HP_dword_409190[0x84 * result];
          if ( Gogoma_HP > 0 )
          {
            gogooma_HP_dword_409190[HP_pos] = Gogoma_HP - 2;
          }
          else
          {
            GogoomaState_DWORD_409194[HP_pos] = 0;
            *((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4];// change flag str
          }
        }
        return result;
      }
      ```

      接下来就要逐步分析*((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4]中各个变量的意义。

      * *((_BYTE *)&FLAG_407028 + result) = FLAG_407028[result]
      * byte_409184很可能是怪物信息相关的，因为之后从0x409194到0x40F8B4都是怪物信息了，所以byte_409184[HP_pos * 4]就是怪物信息中的东西。
        * 0x409184和0x409194都属于第一个怪物信息结构，但对应的成员不同——0x409184不知道代表什么，但确定0x409194对应标志怪物状态的成员。

      1. result

         result来自sub_403440返回值。

         追入这个函数，发现了一个有趣的循环和sub_4027C0函数。

         * 循环如下，这个循环计数器为v3。v3起始于0x408F90，结束于40F6B0，步长值为132，使用的v3[129]。是不是很眼熟？0x408F90 + 0x81就在0x409194附近，而0x40F6B0就在0x40F8B4附近。所以这里操作的也是怪物信息。

           ```c++
           do
             {
               if ( v3[129] && check_shot_4027C0((int)pVoid, &unk_407D84, (int)v3, (int)&v9) && v8 > (double)v9 )
               {
                 v8 = v9;
                 v7 = v1;
                 v2 = 1;
               }
               v3 += 132;
               ++v1;
             }
             while ( (signed int)v3 < (signed int)&flt_40F6B0 );
           ```

         * 进入sub_4027C0函数，就会看到一堆D3DXIntersectTri调用，并且返回值是BOOL，所以sub_4027C0很可能是用于计算位置是否匹配的。查看sub_4027C0引用，发现只出现在了sub_403440中。

         查看result的使用，除了FLAG中，result都是与0x84(132)一起使用的，说明result可能表示是第几个怪物。可以计算一下怪物个数——(0x40F8B4-0x409194)/132 = 50，正好与FLAG字符个数一致，更加确信result表示怪物编号。那么sub_403440可能就是射击位置计算，如果打中怪物则返回怪物编号，没打中就返回-1。

      2. HP_pos = 0x84 * result

         HP_pos = 0x84 * result，所以HP_pos是第result个怪物的某个信息在怪物信息数组中的偏移。

         究竟是什么信息的偏移？查看HP_pos的使用：首先获取一下，如果不为零则-2，为零则处理FLAG对应位置——是不是跟HP的处理很像？所以dword_409190[0x84 * result]就是第result个怪物的HP，当怪物死亡时会进行FLAG[result]^=byte_409184[HP_pos * 4]操作，也就是FLAG[result]^=byte_409184[0x84 * result * 4]。

         ```c++
         Gogoma_HP = gogooma_HP_dword_409190[0x84 * result];
         if ( Gogoma_HP > 0 )
         {
             gogooma_HP_dword_409190[HP_pos] = Gogoma_HP - 2;
         }
         else
         {
             GogoomaState_DWORD_409194[HP_pos] = 0;
             *((_BYTE *)&FLAG_407028 + result) ^= byte_409184[HP_pos * 4];// change flag str
         }
         ```

      3. byte_409184

         查看byte_409184内容，是空的，说明该部分数据需要程序动态加载，需要通过OD加载程序来获取这部分数据。

         但是程序初始化完成后，从OD获取的该部分数据是否是最终操作FLAG时使用的数据呢，也就是说这部分数据会不会随着游戏的展开被程序修改呢？比如怪物每掉一次血就修改一次。查看byte_409184的引用，只有这一个位置使用了byte_409184，所以可能是没有修改过。（*之所以说可能，是因为程序可以通过byte_409183 + 1来修改byte_409184的数据，破解程序很多时候就是尝试，对了就对了，如果错了则说明还有其他情况*）

      至此，就搞清了FLAG的解密过程，用python代码表示如下：

      ```python
      for i in range(0, 50):
      	FLAG_407028[i] = FLAG_407028[i] ^ byte_409184[i * 132 * 4]
      ```

5. 拖进OD，运行程序，把byte_409184-byte_40F8B4的数据dump出来

6. 编写脚本计算flag

   ```python
   #hex_array是一个数组，存储byte_409184-byte_40F8B4的数据
   
   flag_array = [67, 107, 102, 107, 98, 117, 108, 105, 76, 69, 92, 69,
                95, 90, 70, 28, 7, 37, 37, 41, 112, 23, 52, 57, 1, 22, 73, 76, 32, 21,
                 11, 15, 247, 235, 250, 232, 176, 253, 235, 188, 244, 204, 218, 159, 
                 245, 240, 232, 206,240, 169] #从IDA copy的FLAG_407028数组数据
   out_str = ''
   for i in range(0, 50):
       out_str = out_str + chr(hex_array[i * 132 *4] ^ flag_array[i])
   print(out_str)
   ```

## 小结

* 游戏类型的题目的Flag触发条件往往是达到一定的分数或杀光敌人。

  本题压根没有出现分数但出现了clear的字样，所以是杀光敌人的类型。

* 逆向的本质就是各种函数和变量用途的判断，这种判断往往与正向编程相联系

  * 例如：正常情况下FPS游戏失败的判断就是主角HP变为0，因此可以知道0x407020就是主角的HP。

* &是取地址符，&a表示变量a的存储地址，所以&unk_40F8B4表示变量unk_40F8B4的存储地址。

  **注意细节&unk_40F8B4和unk_40F8B4可不一样。**

* IDA也可以用于动态调试，所以本题也可以使用IDA把程序运行起来，然后使用IDAPython计算Flag。

  * 这样就可以直接获取byte_409184-byte_40F8B4的数据，不需要手动处理，脚本如下。

    ```python
    import idaapi
    
    flag = 0x1137028
    
    outstr = ''
    for i in range(0, 50):
        outstr = outstr + chr(Byte(flag +i) ^ Byte(0x1139184 +i * 132 * 4))
    
    print(outstr)
    ```
