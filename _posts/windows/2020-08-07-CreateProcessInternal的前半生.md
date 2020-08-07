---
layout: post
title: "CreateProcessInternal的前半生"
pubtime: 2020-08-02
updatetime: 2020-08-07
categories: Reverse
tags: Windows
---

梳理了windows诸多进程创建函数之间的关系，并对CreateProcessInternalW进行了简单分析，看看系统在调用NtCreateUserProcess之前做了什么。

# CreateProcessInternal的前半生

## windows进程创建函数的关系

在用户层启动windows进程大致可以分为三类：

- NtApi：NtCreateProcess\NtCreateProcessEx\NtCreateUserProcess

- winAPI：CreateProcess系列

- 其他函数：WinExec\system\ShellExecute\ShellExecuteEx\库函数

  (**Zw与Nt的关系**：用户态的Nt和Zw是一样的，ntdll中的Nt函数是Zw函数的别名，两种函数都通过中断进入内核；进入内核后ntoskrnl.dll中的Zw函数调用Nt函数实现功能，Zw函数主要用来将用户态修改为内核态并触发中断(与ntdll中的Zw函数并不同)，而Nt就是中断处理函数。在内核中，previous为用户态时Nt函数将对传递的参数进行严格的检查，而为内核态时则不会，所以为避免额外的参数列表检查从而提高效率，在进行内核驱动开发时要尽量使用Zw函数。(**题外外话**：ntdll中的函数为windows系统原生API，一般正常程序不会直接使用，所以如果某个程序直接调用了ntapi，请怀疑它)

  **ShellExecuteEx**：可以用来以管理员权限启动新进程，lpVerb设置为“runas”。CreateProcess则无法以新的权限启动新进程。)

这三类函数是存在调用关系的：其他函数》winAPI》NtAPI，即所有函数最终都是调用NtApi完成进程创建的，不同的是调用的NtApi。

### NtAPI创建进程

不同版本的系统调用的NtAPI不同（都未公开）：

- 2k：NtCreateProcess 

- xp：NtCreateProcessEx

- vista及之后：NtCreateUserProcess

  使用NtAPI创建进程时，不能直接调用NtCreateProcess和NtCreateProcessEx，需要先用NtOpenFile打开exe文件，用NtCreateSection将文件映射到内存；NtCreateUserProcess对这个过程进行了封装。(*这里目前只学懂了这些*)

### winAPI创建进程

有两个关键函数CreateProcessW和CreateProcessInternalW

CreateProcessW

* 简介

  封装了windows系统内部的进程创建操作，是提供给windows程序开发人员的进程创建接口，也是windows程序员创建进程时最常用的函数

* 地位

  在windows平台，**winAPI以外**用于创建进程的函数（WinExec\system\ShellExecute\ShellExecuteEx等）最终都是通过调用kernel32!CreateProcessW完成进程创建的。*(题外话：因为win平台底层处理用的UNICODE，所以所有的A函数最终都会转到W函数，因此为了提高效率，推荐直接使用W函数)*

  * kernel32!CreateProcessW没有什么操作，会直接转到kernelbase!CreateProcessW（因为kernelbase才是真正代码实现的库），然后直接调用kernelbase!CreateProcessInternalW。

CreateProcessInternalW（未公开）

* 简介

  封装了NtAPI调用的未公开函数，接收并检查外层winAPI传递的参数，构造内层NtAPI的参数并调用NtAPI完成进程创建，最后会进行进程创建后的收尾工作。

* 地位

  win平台下，用户层进程创建的最深层函数，介于winAPI与NtAPI之间，CreateProcess和CreateProcessAsUser最终都会直接调用kernelbase!CreateProcessInternalW，即用户层所有进程创建API最终都是调用kernelbase!CreateProcessInternalW完成的进程创建。

## CreateProcessInternalW内部分析(上)

**起因**

在win7\10直接使用NtCreateUserProcess创建进程遇到了问题，问题记录如下：

* 32位win7：能启动部分程序，其他程序启动无反应（既不报错也没有新进程生成）
* 64位win10，64位程序：能启动部分32位程序，其他程序启动无反应（既不报错也没有新进程生成）
* 64位win10，32位程序：全部都报出0xC000000D的错误
* 另一个64位win10，64位程序：所有程序启动无反应（既不报错也没有新进程生成）
* 另一个64位win10，32位程序：能启动部分程序，但与双击启动效果不同，例如界面大小不同、按钮样式改变、cmd默认路径改变

问题挺多，但应该就是传递的参数有问题，所以尝试分析CreateProcessInternalW看看系统在调用NtCreateUserProcess之前做了什么，尤其是参数的构造。选择了win10下32位的kernelbase进行分析。

**关键数据结构**

从网上收集到NtCreateUserProcess相关的函数原型、数据结构资料。

* NtCreateUserProcess函数原型

  ```c++
  NTSTATUS WINAPI NtCreateUserProcess(
  	OUT PHANDLE ProcessHandle,
  	OUT PHANDLE ThreadHandle,
  	IN ACCESS_MASK ProcessDesiredAccess,
  	IN ACCESS_MASK ThreadDesiredAccess,
  	IN OPTIONAL POBJECT_ATTRIBUTES ProcessObjectAttributes,
  	IN OPTIONAL POBJECT_ATTRIBUTES ThreadObjectAttributes,
  	IN ULONG CreateProcessFlags,
  	IN ULONG CreateThreadFlags,
  	IN OPTIONAL PRTL_USER_PROCESS_PARAMETERS ProcessParameters,
  	IN OUT PPS_CREATE_INFO ProcessCreateInfo,
  	IN OUT OPTIONAL PPS_ATTRIBUTE_LIST AttributeList);
  ```

* 各种结构体

  ```c++
  	typedef struct _UNICODE_STRING
  	{
  		USHORT Length;
  		USHORT MaximumLength;
  		PWSTR  Buffer;
  	} UNICODE_STRING, *PUNICODE_STRING;
  	
  	typedef struct _OBJECT_ATTRIBUTES
  	{
  		ULONG Length;
  		HANDLE RootDirectory;
  		PUNICODE_STRING ObjectName;
  		ULONG Attributes;
  		PVOID SecurityDescriptor;
  		PVOID SecurityQualityOfService;
  	} OBJECT_ATTRIBUTES, *POBJECT_ATTRIBUTES;
  	
  	typedef struct _CURDIR
  	{
  		UNICODE_STRING DosPath;
  		HANDLE Handle;
  	} CURDIR, *PCURDIR;
  	
  	typedef struct _RTL_DRIVE_LETTER_CURDIR
  	{
  		USHORT Flags;
  		USHORT Length;
  		ULONG  TimeStamp;
  		STRING DosPath;
  	} RTL_DRIVE_LETTER_CURDIR, *PRTL_DRIVE_LETTER_CURDIR;
  	
  	typedef struct _RTL_USER_PROCESS_PARAMETERS
  	{
  		ULONG MaximumLength;
  		ULONG Length;
  		ULONG Flags;
  		ULONG DebugFlags;
  		PVOID ConsoleHandle;
  		ULONG ConsoleFlags;
  		HANDLE StandardInput;
  		HANDLE StandardOutput;
  		HANDLE StandardError;
  		CURDIR CurrentDirectory;
  		UNICODE_STRING DllPath;
  		UNICODE_STRING ImagePathName;
  		UNICODE_STRING CommandLine;
  		PVOID Environment;
  		ULONG StartingX;
  		ULONG StartingY;
  		ULONG CountX;
  		ULONG CountY;
  		ULONG CountCharsX;
  		ULONG CountCharsY;
  		ULONG FillAttribute;
  		ULONG WindowFlags;
  		ULONG ShowWindowFlags;
  		UNICODE_STRING WindowTitle;
  		UNICODE_STRING DesktopInfo;
  		UNICODE_STRING ShellInfo;
  		UNICODE_STRING RuntimeData;
  		RTL_DRIVE_LETTER_CURDIR CurrentDirectores[0x20];
  		SIZE_T EnvironmentSize;
  		SIZE_T EnvironmentVersion;
          PVOID PackageDependencyData : Ptr32 Void
         	SIZE_T ProcessGroupId   : Uint4B
     		SIZE_T LoaderThreads    : Uint4B
  
  	} RTL_USER_PROCESS_PARAMETERS, *PRTL_USER_PROCESS_PARAMETERS;
  	
  	typedef struct _PS_CREATE_INFO {
          SIZE_T Size;
          PS_CREATE_STATE State;
          union {
              // PsCreateInitialState
              struct {
                  union {
                      ULONG InitFlags;
                      struct {
                          UCHAR WriteOutputOnExit : 1;
                          UCHAR DetectManifest : 1;
                          UCHAR IFEOSkipDebugger : 1;
                          UCHAR IFEODoNotPropagateKeyState : 1;
                          UCHAR SpareBits1 : 4;
                          UCHAR SpareBits2 : 8;
                          USHORT ProhibitedImageCharacteristics : 16;
                      };
                  };
                  ACCESS_MASK AdditionalFileAccess;
              } InitState;
  
              // PsCreateFailOnSectionCreate
              struct {
                  HANDLE FileHandle;
              } FailSection;
  
              // PsCreateFailExeFormat
              struct {
                  USHORT DllCharacteristics;
              } ExeFormat;
  
              // PsCreateFailExeName
              struct {
                  HANDLE IFEOKey;
              } ExeName;
  
              // PsCreateSuccess
              struct {
  				union {
  					ULONG OutputFlags;
                      struct {
                          UCHAR ProtectedProcess : 1;
                          UCHAR AddressSpaceOverride : 1;
                          UCHAR DevOverrideEnabled : 1; 
                          UCHAR ManifestDetected : 1;
                          UCHAR ProtectedProcessLight : 1;
                          UCHAR SpareBits1 : 3;
                          UCHAR SpareBits2 : 8;
                          USHORT SpareBits3 : 16;
                      };
                  };
                  HANDLE FileHandle;
                  HANDLE SectionHandle;
                  ULONGLONG UserProcessParametersNative;
                  ULONG UserProcessParametersWow64;
                  ULONG CurrentParameterFlags;
                  ULONGLONG PebAddressNative;
                  ULONG PebAddressWow64;
                  ULONGLONG ManifestAddress;
                  ULONG ManifestSize;
              } SuccessState;
          };
      } PS_CREATE_INFO, *PPS_CREATE_INFO;
  	
  	typedef struct _PS_ATTRIBUTE {
          SIZE_T Attribute;
          SIZE_T Size;
          union {
              ULONG_PTR Value;
              PVOID ValuePtr;	
          };
          SIZE_T ReturnLength;
      } PS_ATTRIBUTE, *PPS_ATTRIBUTE;
  	
      typedef struct _PS_ATTRIBUTE_LIST {
          SIZE_T TotalLength;					/// sizeof(PS_ATTRIBUTE_LIST)
          PS_ATTRIBUTE Attributes[20];		/// Depends on how many attribute entries should be supplied to NtCreateUserProcess
      } PS_ATTRIBUTE_LIST, *PPS_ATTRIBUTE_LIST;
  ```

**分析**

CreateProcessInternalW函数主要做了三件事：构造参数、调用NtCreateProcess、收尾。但是函数细节相当复杂，有一些地方我不能完全明白是什么意思，现在把明白的部分记录下来。

1. 首先通过CreateProcessW查看CreateProcessInternalW的参数，CreateProcessW伪代码如下：

   ```c++
   BOOL __stdcall CreateProcessW(LPCWSTR lpApplicationName, LPWSTR lpCommandLine, LPSECURITY_ATTRIBUTES lpProcessAttributes, LPSECURITY_ATTRIBUTES lpThreadAttributes, BOOL bInheritHandles, DWORD dwCreationFlags, LPVOID lpEnvironment, LPCWSTR lpCurrentDirectory, LPSTARTUPINFOW lpStartupInfo, LPPROCESS_INFORMATION lpProcessInformation)
   {
     return CreateProcessInternalW(
              0,
              lpApplicationName,
              lpCommandLine,
              lpProcessAttributes,
              lpThreadAttributes,
              bInheritHandles,
              dwCreationFlags,
              lpEnvironment,
              lpCurrentDirectory,
              lpStartupInfo,
              lpProcessInformation,
              0);
   }
   ```

   可见CreateProcessInternalW在参数上就比CreateProcessW多了开头结尾的两个0，其余参数与CreateProcessW一致。

2. CreateProcessInternalW首先进行了局部变量的初始化。

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-27-30.png)

3. 检查关键参数是否为空

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-30-14.png)

4. 检查CreateFlags标志

   有些标志不能同时存在，同时需要根据标志设置优先级和调试对象。

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-31-32.png)

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-33-01.png)

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-34-11.png)

5. 构建NtCreateUserProcess的ProcessCreateFlags参数（虽然之后可能会有调整，但这里设置的标志基本就是最终使用的标志了）

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-36-17.png)

6. 构建NtCreateUserProcess的AttributesList参数（虽然之后可能还会添加，但这里基本就是最终使用的参数了）

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-37-28.png)

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-49-02.png)

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_11-49-46.png)

   如果为CreateProcess传入的CreateFlags为0，则这里的if都为假。

7. 清空ProcessInfo

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_12-36-18.png)

   准备环境描述块（当条件成立时）

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_12-35-44.png)

8. 根据CreateFlags标志对ProcessCreateFlag和AttributesList进行调整（一般不会用到）

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_12-38-54.png)

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_12-39-41.png)

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_12-41-14.png)

9. 结构化两个POBJECT_ATTRIBUTES参数（实际使用时这两个参数设为NULL就可以）

   ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_12-42-23.png)

10. 接下来会进入一个死循环，在这个死循环中完成了对ProcessParameters的构建和NtCreateUserProcess的调用。*（不明白这里为什么会用死循环）*

11. 这个死循环中首先会对循环中用到的变量进行常规操作，即清零。然后就会去处理cmdLine和ImageName这两个字符串，主要就是将字符串规范化、检查路径是否合理、构造绝对路径，流程如下：

    ![](https://chrishuppor.github.io/image/2020-8-3-17-22.png)

12. 接下来有一片代码不知道在做什么，函数要不没有符号要不就查不到，猜测可能是根据PEB中的某些信息进行环境变量的设置。*（这部分可能是常规操作，但我见识少，所以就搞不懂了）*

    使用测试程序进行测试：测试程序使用CreateProcessW创建一个带参进程，用OD跟踪，发现这部分不在路径中。

13. 跟会添加新的变量到AttributesList，仍旧没有看出来这里的判断是什么，但同样经过OD调试发现，这里的条件是不成立的。

14. 接着就是构造ProcessParameters参数

    ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_17-36-54.png)

    这里的CreateParams_100E25E7不是导出函数，也没有符号，内部调用了RtlCreateProcessParametersEx。

    ```c++
    NTSTATUS __fastcall RtlCreateProcessParametersEx(
            PRTL_USER_PROCESS_PARAMETERS pProcessParameters,
            PUNICODE_STRING  ImagePathName,
            PUNICODE_STRING  DllPath,
            PUNICODE_STRING  CurrentDirectory
            PUNICODE_STRING  CommandLine,
            PWSTR            Environment,
            PUNICODE_STRING  WindowTitle,
            PUNICODE_STRING  DesktopInfo,
            PUNICODE_STRING  ShellInfo,
            PUNICODE_STRING  RuntimeData
            DWORD           UNKONW);
    ```

15. 在调用之前还会添加一个新的变量到AttributesList，并且计算AttributesList的大小填充到AttributesList的TotalLength中。经OD追踪，发现AttributesList一般是包含5个Attributes结构体

    ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_17-39-26.png)

16. 最终，调用NtCreateUserProcess，前半生结束。

    ![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_17-42-25.png)

    前两个参数是输出的PHANDLE，三四是写死的0x2000000，ProcessObjectAttributes和ThreadObjectAttributes为NULL，varFinalCreateFlag从OD得到是0x200，PsCreateInfo是输出参数，所以关键参数就是ProcessParameters和PsAttributeList。

**流程**

* 前半生整体流程

![](https://chrishuppor.github.io/image/Snipaste_2020-08-03_17-49-43.png)

* 子流程

  * CreateFlags处理，构造NtCreateUserProcess创建标志参数

    黄色块是CreateFlags为0时的执行路径。

    ![](https://chrishuppor.github.io/image/2020-8-3-17-53.png)

  * 构造参数PS_ATTRIBUTE_LIST

    黄色块是执行路径。

    ![](https://chrishuppor.github.io/image/2020-8-3-17-53.png)

  * 检查cmdLine和ImageName

    ![](https://chrishuppor.github.io/image/2020-8-3-17-22.png)

[[IDB和vsdx](https://github.com/chrishuppor/attachToBlog/tree/master/CreateProcessInternalW.zip)]

---

尽管分析的并不十分清楚，但仍然感觉很吃力，尤其是面对一堆不知道做什么用的函数时。

我觉得主要原因有三个：学的少、练习少、资料少。无论如何，菜是原罪，以后更要多学多看多动手。