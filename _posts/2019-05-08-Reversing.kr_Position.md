---
layout: post
title: "Reversing.kr_Position"
date: 2019-5-8 10:48:10
categories: WriteUp
tags: Reversing_kr
---

微简RE challenge网站，my_4ear_3hr1s 就系我啦，第7题解题过程及题后思考记录如下。

# Reversing.kr_Position

## 解题过程

1. 阅读readme

   这是一个验证码程序，要求找出指定serial对应的name。

2. 运行程序

   有两个输入框，分别输入name和serial。验证失败时下方会显示wrong，猜测验证通过时会修改下方字符串。

3. 拖进PEiD、virustotal没有什么特别的。拖进RH也没有发现什么，就是正常的MFC程序，输入使用的edit控件，wrong使用static控件显示。

4. 拖进PEview，查看IAT找出程序可能使用的获得输入的API。

   mfc100u.dll中使用序号导入函数，需要使用dependency查看mfc100u.dll来一一查询。

5. 拖进OD

   1. 搜索字符串

      如图，很容易就发现了提示信息 correct和wrong。

      ![图1 搜索字符串](https://chrishuppor.github.io/image/Snipaste_2019-05-08_08-48-40.PNG)

   2. 转到wrong的引用，查看周围代码。

      如图，这是一个结构清晰的分支代码：如果验证成功就显示correct，不成功就跳转到显示wrong。

      ![图2 wrong的引用](https://chrishuppor.github.io/image/Snipaste_2019-05-08_08-50-04.PNG)

      查看jz之前的代码，发现是在call 00BA1740之后进行jz的，所以这个函数很可能是验证码比对函数。

   3. 查看函数00BA1740

      这个函数有些复杂，转到IDA尝试查看反编译代码。

6. 拖进IDA

   1. 查看sub_401740（这个就是之前的00BA1740函数，因为加载机制不一样，所以绝对地址不同，相对基址的地址都是1740）

      一言难尽，有很多奇怪的东西，又有很多数学运算，是验证码对比函数无疑了，接下来自顶向下逐块分析。

      1. 首先创建了三个CString对象。通过后续代码可知这三个对象分别用于存储name、serial和中间变量

         ```c++
         ATL::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>(&v50_name);
         v1 = 0;
         v53 = 0;
         ATL::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>(&v51_serial);
         ATL::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>::CStringT<wchar_t,StrTraitMFC_DLL<wchar_t,ATL::ChTraitsCRT<wchar_t>>>(&v52_tmp);
         ```

      2. 接下来获取了一个edit的字符串

         ```c++
         CWnd::GetWindowTextW(a1_this + 304, &v50_name);
         ```

         因为一共有两个edit，所以有两个GetWindowTextW，分别获取的name和serial。究竟哪个是name，哪个是serial，可以通过获取的控件ID或对获取字符串的处理来判断。通过之后的代码可以判断出后一个是serial，那么前面这个就是name了。

         ```c++
         if ( *(_DWORD *)(v50_name - 12) == 4 )
         ```

         name的长度判断，要求name长为4。

      3. 接下来是一个循环

         这个循环看起来很大，其实主要是第一层的if内容较多，把if折叠就好看很多了。

         ```
         while ( (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) >= 'a'
                  && (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) <= 'z' )
          {                                           // name是小写字母
               if ( ++name_count_1 >= 4 ) {...}
          }
         ```

         如图，当name_count_1 < 4时就相当于：

         ```c++
         while ( (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) >= 'a'
                  && (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, name_count_1) <= 'z' )
          {                                           // name是小写字母
               ++name_count_1 >= 4;
          }
         ```

         当name_count_1 = 4时就会进入if结构，然后从if中直接跳出循环。

         所以这个循环的目的就是检查v50_name是不是由小写字符组成，检查的长度是4，说明name是一个由小写字母组成的长度为4的字符串。

      4. if ( ++name_count_1 >= 4 )内部首先是一段故伎重演的循环

         ```c++
         LABEL_7:
         	v4 = 0; 
         while ( 1 )
         {
         	if ( v1 != v4 )//逐个查看v50_name[v4]是否与v50_name[v1]相同
         	{
         		v5 = ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, v4);// v5 = name[v4]
         		if ( (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v50_name, v1) == v5 )
         			goto LABEL_2;                     // 要求name的四个字符互不相同
         	}
         	if ( ++v4 >= 4 )
         	{
         		if ( ++v1 < 4 )//当v4 = 4, v1 < 4时，重置v4，再比较一遍v50_name[v1]和v50_name其他字符
         			goto LABEL_7;
         		...
         	}
         	...
         }
         ```

         这段代码用于比较name中的字符，要求name的四个字符互不相同。

      5. 当name通过检查后，才会进行serial的获取

         ```c++
         CWnd::GetWindowTextW(a1_this + 420, &v51_serial);
         ```

         然后是serial的格式验证：

         ```c++
         if ( *(_DWORD *)(v51_serial - 12) == 11
                       && (unsigned __int16)ATL::CSimpleStringT<wchar_t,1>::GetAt(&v51_serial, 5) == '-' )// 要求serial格式xxxxx-xxxxx
         ```

         要求serial是一个长为11的字符串，且serial[5] =='-'。这与readme给出的serial格式一致，也是因此判断这里是serial，上面那个是name。

      6. serial格式验证通过后就是通过name计算出结果，将结果与serial比对的过程。

         这部分有两个关键点：

         * 知道GetAt(a, pos) = a[pos]
         * 明白(a >> count )&1表示取a的第count+1位，例如 ((name_0 >> 4) & 1) 是取name_0从右向左第5个比特
         * 知道itow_s(v, tmp, 0xAu, 10)表示将v转化为十进制，并以字符串的形式存储在tmp中，tmp最大长度为0xA。
         * 小写字母的都是的ascii码都是11xxxxx，所以没有计算第6、7位，逆回去的时候也不需要计算。

         这样就能得到正向计算公式：

         ```
         name[0]0 + name[1]2 + 6 = serial[0]
         name[0]3 + name[1]3 + 6 = serial[1]
         name[0]1 + name[1]4 + 6 = serial[2]
         name[0]2 + name[1]0 + 6 = serial[3]
         name[0]4 + name[1]1 + 6 = serial[4]
         
         name[2]0 + name[3]2 + 6 = serial[6]
         name[2]3 + name[3]3 + 6 = serial[7]
         name[2]1 + name[3]4 + 6 = serial[8]
         name[2]2 + name[3]0 + 6 = serial[9]
         name[2]4 + name[3]1 + 6 = serial[10]
         ```

         已知serial，从而可以知道：

         ```c++
         name[0]0 + name[1]2 = 1 
         name[0]3 + name[1]3 = 0
         name[0]1 + name[1]4 = 2
         name[0]2 + name[1]0 = 1
         name[0]4 + name[1]1 = 0
         
         name[2]0 + name[3]2 = 1
         name[2]3 + name[3]3 = 1
         name[2]1 + name[3]4 = 1
         name[2]2 + name[3]0 = 1
         name[2]4 + name[3]1 = 0
         ```

         已知name[3] = 'p'，所以易得name[2] = 'm'
         因为单个bit只能是0或1，所以——结果为1，说明两个比特一个为0一个为1；结果为0，说明二者都为0；结果为2，说明二者都为1。由此可以推出 name[0] = 01100x1x,name[1] = 01110x0x 

         使用py计算一下可以得到四个结果，其中一个因为要求name四个字符互不相同而去掉，另外三个都可以，其中一个就是Flag。

## 所遇问题

1. **问题：**因为缺少mfc100u.dll和msvcr100.dll无法运行程序，所以先为分析机安装了这两个dll。之后又会说无法定位输入点xxx与mfc100u.dll，网上说是dll版本与机器不和，装个vs2010就可以了。

   **解决方案：**这个程序没有毒，我的主机又装有vs，所以最终决定在主机进行动态分析。

2. **问题：**拖进IDA后，发现有很多函数，根本不知道要看哪一个。MFC是事件驱动的，不像CUI程序有一个main入口，之后一切由main展开。

   **解决方案：**把程序拖入OD，通过断点或其他来定位要查看的函数地址，然后回到IDA中查看该函数。

3. **问题：**目标函数中使用了模板类，而且通过偏移地址来获取类成员的值，不容易看出这个值的用途。

   **解决方案：**额，慢慢查，耐心的看算不算解决办法...其实明白那些奇怪的偏移是在获取类成员变量的值就一切好说了，例如```if ( *(_DWORD *)(v50_name - 12) == 4 )  ```

## 小结

* 这是个KeyGen的程序，模式是通过用户输入计算出serial，本质上与第二题没有区别。之前也说了，这种程序破解的关键在于获取计算算法的逆算法，这个也是破解难度的关键。
* 需要熟悉对整数的位操作
  * tmp & 1表示取tmp的最低位，当与(tmp >> count)配合时就可以获取tmp的第(count+1)位。
* 破解MFC程序时，可以先通过OD找到关键函数，然后通过IDA分析函数的反编译代码。