---
layout: post
title: "GetLastError"
pubtime: 2019-04-18
updatetime: 2019-04-18
categories: LearningNote
tags: Windows
---

GetLastError函数的原理和使用。

# 1 GetLastError

## 1.1 原理

当一个windows函数运行失败后，会将一个32位的错误代码存储在**线程本地存储器**，直到下一个windows函数运行失败后才会被修改。

* 一个线程保存一个错误代码，不同的线程有不同的错误代码
* 多个windows函数运行失败，只保留最后一个错误代码

## 1.2 使用

声明

* 微软定义的错误代码声明在WinError.h中

* 用户可以按照如下规则自行定义一个32位错误代码

  ![](https://chrishuppor.github.io/image/Snipaste_2020-04-18_17-58-53.png)

使用

* 在代码中：可以使用GetLastError()来获取错误代码，使用SetLastError(DWORD)来指定错误代码(多用于自定义函数报告错误信息)，使用FormatMessage可以将错误码转换为描述语句。

* 在IDE中：如图，在监视器中添加一行“@err,hr”

  ![](https://chrishuppor.github.io/image/Snipaste_2020-04-18_17-44-15.png)

* 在外部：使用vs提供的errlook小工具查询错误码

  ![](https://chrishuppor.github.io/image/Snipaste_2020-04-18_17-55-32.png)

##1.3 在程序中获取错误码描述语句

关键函数：GetLastError、FormatMessage

示例代码：

```c
/***********************************************
功能：将错误代码转换为文字，以提示框的形式显示出来
参数：errStr : 自定义的标题
***********************************************/
//v1.0:固定mes大小
void ThrowMes(TCHAR *errStr) {
	TCHAR mes[1024];
	int err = GetLastError();
	FormatMessage(FORMAT_MESSAGE_FROM_SYSTEM, NULL, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), mes, 1024, NULL);

	MessageBox(NULL, mes, errStr, 0);
	_tprintf(TEXT("%s:%s\n"), errStr, mes);
}

//v2.0:动态mes大小
void ThrowMes(CONST TCHAR *errStr) {
	LPTSTR lpmes = NULL;
	int err = GetLastError();

	//FORMAT_MESSAGE_FROM_SYSTEM表示从system message-table resource(s)获取mes
	//FORMAT_MESSAGE_ALLOCATE_BUFFER表示要为lpmes创建空间
	//err:错误码
	//MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT):语言ID
	//lpmes:指向存放mes的内存空间，需要LocalFree手动释放
	//0:lpmes指向的空间最小字符串长度为0
	BOOL bOk = FormatMessage(FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_ALLOCATE_BUFFER, NULL, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPTSTR)&lpmes, 0, NULL);

	if(bOk == FALSE){
		// Is it a network-related error?
		HMODULE hDll = LoadLibraryEx(TEXT("netmsg.dll"), NULL,
			DONT_RESOLVE_DLL_REFERENCES);

		if (hDll != NULL) {
			bOk = FormatMessage(FORMAT_MESSAGE_FROM_HMODULE|FORMAT_MESSAGE_IGNORE_INSERTS|FORMAT_MESSAGE_ALLOCATE_BUFFER,
				hDll, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
				(LPTSTR)&lpmes, 0, NULL);
			FreeLibrary(hDll);
		}
	}
	if (bOk && lpmes != NULL) {
		MessageBox(NULL, lpmes, errStr, 0);
		LocalFree(lpmes);
	}
	else
		MessageBox(NULL, TEXT("ErrCode Not Found."), errStr, 0);

}
```
FormatMessage参数简要说明

* 第一个参数：

  * FORMAT_MESSAGE_FROM_SYSTEM 表示从system message-table resource(s)查找描述语句
  * FORMAT_MESSAGE_ALLOCATE_BUFFER 表示要为返回的描述语句创建空间

* 第三个参数：

  * 错误码(如果第一个参数是FORMAT_MESSAGE_FROM_STRING就被忽略)

* 第四个参数

  * 语言ID：该ID与windows系统的LANGUAGE ID是一致的，所以可以直接使用windows对应的language ID号码。

  * MAKELANGID()：是对计算语言ID的宏，根据LANG_NEUTRAL（主号）, SUBLANG_DEFAULT（子号）来构建语言ID。

    | language | ID   |
    | -------- | ---- |
    | 英语     | 1033 |
    | 简体中文 | 2052 |
    | 繁体中文 | 1028 |

    （ID详情请参考<http://www.cnblogs.com/wangweixf/archive/2008/08/15/1268537.html>）

* 第五个参数：

  * 指向描述语句存放空间的指针，需要LocalFree手动释放

（FormatMessage详情请参考<https://docs.microsoft.com/zh-cn/windows/win32/api/winbase/nf-winbase-formatmessage>）