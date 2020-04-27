---
layout: post
title: "Reversing.kr_WindowKernel"
pubtime: 2019-5-21 12:24:46
updatetime: 2019-5-21 12:24:46
categories: Reverse
tags: WriteUp
---

Reversing.kr的第十二题，是一个与驱动有关的题目，如果对驱动比较熟悉就是它左边那个题的难度，否则就是它上边那个题的难度。

# Reversing.kr_WindowKernel

## 解题过程

1. 运行程序
   1. 拖进64位机器，运行exe，提示无法创建服务。以管理员权限运行，提示无法启动服务。很明显，服务程序是sys，如果服务启动失败就是sys的安装问题。
   2. sys安装失败，有可能是位数不合适。查看sys位数，果然是32位。
   3. 拖进32位机器，以管理员权限运行exe，显示出提示信息"Analyze me! :>Hit)Keyboard"。
   4. 下方是一个不可输入的文字框，旁边有一个enable的按钮。点击按钮，发现按钮文字变成了check，文字框也变得可以输入了。输入几个字符，点击按钮，弹出提示框wrong，并恢复到enable状态。

2. 将exe拖入IDA

   1. 窗口函数是自定义的DialogFunc。

      通过查询可知DialogFunc参数宏定义在WinUser.h中，查看其宏定义可知，272表示WM_INITDIALOG，273表示WM_COMMAND，275表示WM_TIMER，a3 == 2表示WM_DESTROY 。没有看到1002和1003相关，可能是自定义的消息。

      ```c
      BOOL __stdcall DialogFunc(HWND hWnd, UINT a2, WPARAM a3, LPARAM a4)
      {
        if ( a2 == 272 )//WM_INITDIALOG
        {
          SetDlgItemTextW(hWnd, 1001, L"Wait.. ");
          SetTimer(hWnd, 0x464u, 0x3E8u, 0);
          return 1;
        }
        if ( a2 != 273 )//not wm_command
        {
          if ( a2 == 275 )// WM_TIMER
          {
            KillTimer(hWnd, 0x464u);
            sub_401310(hWnd);
            return 1;
          }
          return 0;
        }
        if ( (unsigned __int16)a3 == 2 ) //WM_DESTROY 
        {
          SetDlgItemTextW(hWnd, 1001, L"Wait.. ");
          sub_401490();
          EndDialog(hWnd, 2);
          return 1;
        }
        if ( (unsigned __int16)a3 == 1002 )
        {
          if ( a3 >> 16 == 1024 )
          {
            Sleep(0x1F4u);
            return 1;
          }
          return 1;
        }
        if ( (unsigned __int16)a3 != 1003 )
          return 0;
        sub_401110(hWnd);
        return 1;
      }
      ```

      DialogFunc逻辑很清晰：创建对话框的时候添加一个定时器，当超时时创建并启动服务程序，服务由WinKer.sys来提供。程序结束时关闭服务程序。当a3位1003时，调用sub_401110函数。

   2. 查看sub_401110

      如下，发现了最后的提示信息```MessageBoxW(v1, L"Correct!", L"Reversing.Kr", 0x40u);```，说明这个函数就是check相关核心函数。

      ```c
      HWND __thiscall sub_401110(HWND hDlg)
      {
        v1 = hDlg;
        GetDlgItemTextW(hDlg, 1003, &String, 512);
        if ( lstrcmpW(&String, L"Enable") )
        {
          result = (HWND)lstrcmpW(&String, L"Check");
          if ( !result )
          {
            if ( sub_401280(0x2000) == 1 )
              MessageBoxW(v1, L"Correct!", L"Reversing.Kr", 0x40u);
            else
              MessageBoxW(v1, L"Wrong", L"Reversing.Kr", 0x10u);
            SetDlgItemTextW(v1, 1002, &word_4021F0);
            v5 = GetDlgItem(v1, 1002);
            EnableWindow(v5, 0);
            result = (HWND)SetDlgItemTextW(v1, 1003, L"Enable");
          }
        }
        else if ( sub_401280(4096) )
        {
          v3 = GetDlgItem(v1, 1002);
          EnableWindow(v3, 1);
          SetDlgItemTextW(v1, 1003, L"Check");
          SetDlgItemTextW(v1, 1002, &word_4021F0);
          v4 = GetDlgItem(v1, 1002);
          result = SetFocus(v4);
        }
        else
        {
          result = (HWND)MessageBoxW(v1, L"Device Error", L"Reversing.Kr", 0x10u);
        }
        return result;
      }
      ```

      1. 如图，使用RH查看exe资源文件，1003正是按钮的ID。

         ![图1 Dialog资源ID查看](https://chrishuppor.github.io/image/Snipaste_2019-05-21_09-15-14.PNG)

         所以```GetDlgItemTextW(hDlg, 1003, &String, 512);```获取的是按钮显示的文字。

      2. 如下，如果按钮是check且sub_401280(0x2000) == 1，则表示验证通过，否则恢复enable状态。

         ```c
         if ( lstrcmpW(&String, L"Enable") )
         {
             result = (HWND)lstrcmpW(&String, L"Check");
             if ( !result )
             {
                 if ( sub_401280(0x2000) == 1 )
                     MessageBoxW(v1, L"Correct!", L"Reversing.Kr", 0x40u);
                 else
                     MessageBoxW(v1, L"Wrong", L"Reversing.Kr", 0x10u);
                 SetDlgItemTextW(v1, 1002, &word_4021F0);
                 v5 = GetDlgItem(v1, 1002);
                 EnableWindow(v5, 0);
                 result = (HWND)SetDlgItemTextW(v1, 1003, L"Enable");
             }
         }
         ```

      3. 如下，如果sub_401280(0x1000)为真则变为check状态。

         ```c
         else if ( sub_401280(0x1000) )
         {
             v3 = GetDlgItem(v1, 1002);
             EnableWindow(v3, 1);
             SetDlgItemTextW(v1, 1003, L"Check");
             SetDlgItemTextW(v1, 1002, &word_4021F0);
             v4 = GetDlgItem(v1, 1002);
             result = SetFocus(v4);
         }
         ```

      4. 由此可见，sub_401280十分关键，当参数为0x1000时控制exe变换状态，当参数为0x2000时给出输入是否正确的判断。

         如下，可知0x1000和0x2000的参数名为dwIoControlCode，会进一步传递给DeviceIoControl函数。DeviceIoControl返回数据存储在OutBuffer中，并作为sub_401280返回值。

         ```c
         int __usercall sub_401280@<eax>(HWND a1@<edi>, DWORD dwIoControlCode)
         {
             v2 = CreateFileW(L"\\\\.\\RevKr", 0xC0000000, 0, 0, 3u, 0, 0);
             if ( v2 == (HANDLE)-1 )
             {
                 MessageBoxW(a1, L"[Error] CreateFile", L"Reversing.Kr", 0x10u);
                 result = 0;
             }
             else if ( DeviceIoControl(v2, dwIoControlCode, 0, 0, &OutBuffer, 4u, &BytesReturned, 0) )
             {
                 CloseHandle(v2);
                 result = OutBuffer;
             }
             else
             {
                 MessageBoxW(a1, L"[Error] DeviceIoControl", L"Reversing.Kr", 0x10u);
                 result = 0;
             }
             return result;
         }
         ```

         通过查询可知，DeviceIoControl函数用于“Sends a control code directly to a specified device driver, causing the corresponding device to perform the corresponding operation.”。

         经查询可知，dwIoControlCode是硬件操作码，可以自定义，由```#define CTL_CODE(DeviceType, Function, Method, Access) (((DeviceType) << 16) | ((Access) << 14) | ((Function) << 2) | (Method))```来定义一个硬件操作码，对应的宏定义在Windev.h中。但是搜了半天也没找到Windev.h，结果找到了WinIoCtl.h，这里面也有对应的宏定义。从而可知0x1000和0x2000只有Function字段有值，且function是作者自定义的。

         至此，题目的结构就清晰了——使用exe将sys注册为服务并启动，定义了0x1000和0x2000两个硬件操作码，使用DeviceIoControl函数控制驱动接收用户输入并进行比较。

   3. 动态调试sys需要配置双机调试环境，比较麻烦，因此先尝试使用IDA静态调试。将sys拖入IDA。

      1. 查看string，没有什么特别的，只好从DriveEntry看起。

         入口一共两个函数：sub_14005和sub_11466。其中，sub_14005显然跟主功能没有关系。经查询， BugCheckParameter2与系统蓝屏有关，那么就与我们要找的无关咯。所以关键在sub_11466中

      2. 查看sub_11466

         看样子像一个init函数，负责各个全局参数的初始化以及事件与函数的绑定。

         对比一个简单的驱动编写程序可知之前的猜测是正确的。如下，一共有五处函数绑定。

         ```c
         DriverObject->DriverUnload = (PDRIVER_UNLOAD)sub_1131C;
         
         DriverObject->MajorFunction[14] = (PDRIVER_DISPATCH)sub_11288;
         DriverObject->MajorFunction[0] = (PDRIVER_DISPATCH)sub_112F8;
         DriverObject->MajorFunction[2] = (PDRIVER_DISPATCH)sub_112F8;
         KeInitializeDpc(&DeviceObject->Dpc, sub_11266, DeviceObject);
         ```

         通过查询DriverObject可知，这是一个结构体，用于描述驱动运行所需要的一些数据。其成员DriverUnload用于存放"The entry pointer for the driver's Unload routine"；MajorFunction是一个数组，用于为派遣动作设定对应的派遣函数，其中MajorFunction的下标为代表IRP major function code的IRP_MJ_XXX样式的宏，定义在wdm.h中。查询DeviceObject->Dpc可知，这是一个设备对象的延迟过程调用（DPC）对象，内部以一个灵活的定时器，用于处理设备的异步操作。

         网上搜索驱动开发用到的派遣函数序号(wdm.h)，可知MajorFunction[14]表示IRP_MJ_DEVICE_CONTROL，MajorFunction[0]表示IRP_MJ_CREATE，MajorFunction[2]表示IRP_MJ_CLOSE。

         因此，sub_1131C是用于驱动卸载的，sub_112F8是用于初始化和关闭的，所以都不是我们关心的。sub_11288是设备控制函数，是分析的重点。sub_11266是设备异步操作函数，也是我们要关注的。

      3. 查看sub_11288

         尽管不知道Irp->Tail.Overlay.PacketType + 12是什么，但根据0x1000和0x2000可以猜到这个就是dwIoControlCode。当控制码为0x1000时进行初始化，当控制码为0x2000时的设置表明dword_13030很可能是一个flag，*(_DWORD *)&v3->Type很可能是返回值。

         IofCompleteRequest用于消息传递，类似于DialogFunc中的Msg传递函数。

         因此，本函数的功能是根据控制码进行配置IRP结构，然后进行消息传递。

         ```c
         int __stdcall sub_11288(int a1, PIRP Irp)
         {
             int v2; // edx@1
             struct _IRP *v3; // eax@1
         
             v2 = *(_DWORD *)(Irp->Tail.Overlay.PacketType + 12);
             v3 = Irp->AssociatedIrp.MasterIrp;
             if ( v2 == 0x1000 )
             {
                 *(_DWORD *)&v3->Type = 1;
                 dword_13030 = 1;
                 dword_13034 = 0;
                 dword_13024 = 0;
                 dword_1300C = 0;
             }
             else if ( v2 == 0x2000 )
             {
                 dword_13030 = 0;
                 *(_DWORD *)&v3->Type = dword_13024;
             }
             Irp->IoStatus.Status = 0;
             Irp->IoStatus.Information = 4;
             IofCompleteRequest(Irp, 0);
             return 0;
         }
         ```

         既然进行了消息传递，那么肯定有消息处理函数。是不是想到了之前的Dpc函数sub_11266？这个函数大概就是用于进行消息处理的——根据不同的消息进行不同的设备操作。

      4. 查看sub_11266

         本函数首先使用READ_PORT_UCHAR指定从0x60端口读取一字节数据，然后调用 sub_111DC函数处理这一字节的数据。其中，READ_PORT_UCHAR的功能很好查，通过搜索windows设备端口0x60得知，这个端口表示键盘。

         因此，本函数的功能就是从键盘读取一个字节，然后处理它，处理函数为 sub_111DC。

      5. 查看 sub_111DC

         如下，首先判断dword_1300C是否是1，如果不是1则会判断dword_13034的值，根据dword_13034值的不同有不同的操作，总结来说就是dword_13034为偶数时仅++，对a1不做处理；dword_13034为奇数时要对a1进行比较，如果不通过则将dword_1300C置为1。如果通过比较，当dword_13034为1、3、5时直接++，当dword_13034为7时则加100。如果没有dword_13034对应的值，则使用sub_11156来处理a1。

         ```c
         signed int __stdcall sub_111DC(char a1)
         {
             result = 1;
             if ( dword_1300C != 1 )
             {
                 switch ( dword_13034 )
                 {
                     case 0:
                     case 2:
                     case 4:
                     case 6:
                         goto LABEL_3;
                     case 1:
                         v2 = a1 == 0xA5u;
                         goto LABEL_6;
                     case 3:
                         v2 = a1 == 0x92u;
                         goto LABEL_6;
                     case 5:
                         v2 = a1 == 0x95u;
         LABEL_6:
                         if ( !v2 )
                             goto LABEL_7;
         LABEL_3:
                         ++dword_13034;
                         break;
                     case 7:
                         if ( a1 == 0xB0u )
                             dword_13034 = 100;
                         else
         LABEL_7:
                         dword_1300C = 1;
                         break;
                     default:
                         result = sub_11156(a1);
                         break;
                 }
             }
             return result;
         }
         ```

      6. 查看sub_11156

         发现其逻辑与sub_111DC一致，只不过在比较前对a1进行了异或——a1 = a1 ^ 0x12。如果没有dword_13034对应的值，则使用sub_110D0来处理a1。

      7. 查看sub_110D0

         发现其逻辑与sub_111DC和sub_11156一致，只不过在比较前对a1进行了异或——a1 = a1 ^ 0x5。如果没有dword_13034对应的值，则返回dword_13034-200。

      8. 将sub_111DC、sub_11156、sub_110D0连起来看就可以发现，这三个函数接力进行输入字符匹配，dword_13034则类似于一个计数器，高位用于记录是第几个处理函数，低位记录是本函数处理的第几个字符。

         之所以只处理奇数，是因为一次按键会产生两个字符，第一个字符表示按下去的按键，第二个字符表示弹起的键。虽然都是一个键，但硬件处理时要标记是弹起还是按下，所以有两个值，且按下的值在前。但是实际匹配时只需要匹配一个就行了，本题选择匹配弹起的键的值。

         因此，一共有12个字符，如果都匹配则将retn_13024设为1，否则设为0。

         根据匹配逻辑可知这十二个字符如下，注意第三个函数接收到的参数已经经过0x12异或了。

         ```python
         f = [0xA5, 0x92, 0x95, 0xb0,
              0x12^0xB2, 0x12^0x85, 0x12^0xA3, 0x12^0x86,
               0x12^0xB4^0x5, 0x12^0x5^0x8F, 0x12^0x5^0x8F, 0x12^0x5^ 0xB2]
         ```

      9. 因为这十二个字符是通过端口从键盘IO中读取的，所以这些不是ascii，而是按键的扫描码，而且是按键弹起的扫描码。需要自行对应ascii码，然后将ascii数值转换为chr.

         脚本如下：

         ```python
         ScanCode = [0X90, 0X91, 0X92, 0X93, 0X94, 0X95, 0X96, 0X97, 0X98, 0X99,
                     0x9e, 0x9f, 0xa0, 0xa1, 0xa2, 0xa3, 0xa4, 0xa5, 0xa6,
                     0xac, 0xad, 0xae, 0xaf, 0xb0, 0xb1, 0xb2]
         scanChr = 'qwertyuiopasdfghjklzxcvbnm'
         
         f = [0xA5, 0x92, 0x95, 0xb0,
              0x12^0xB2, 0x12^0x85, 0x12^0xA3, 0x12^0x86,
               0x12^0xB4^0x5, 0x12^0x5^0x8F, 0x12^0x5^0x8F, 0x12^0x5^ 0xB2]
         flagStr = ''
         for item in f:
             flagStr += (scanChr[ScanCode.index(item)])
         
         print(flagStr)
         ```

## 小结

就像开头说的一样，做题人如果熟悉甚至仅仅是编写过驱动，那么就会觉得这个题和EasyELF一样，纯靠读IDA代码就能破解。如果一点不了解驱动也没关系，只要耐心的在网上搜索，最终也能破解。

* 在使用IDA查看函数反编译代码时，一些参数常常直接显示数值而不是宏名称，这时候可以先查这个函数的参数，一般MSDN都会给出函数所在头文件，然后查看对应的头文件，从中找到值对应的宏。
* [键盘扫描码（他人博文）](https://blog.csdn.net/qq_37232329/article/details/79926440)