---
layout: post
title: "IDApython安装"
pubtime: 2018-04-07
updatetime: 2018-04-07
categories: Reverse
tags: IDAPython Tools
---

安装IDApython，高版本的IDA自带IDApython。本文章非原创，在原文基础上进行的修改。[原文地址](http://www.cnblogs.com/blacksunny/p/7215247.html)

# 注意事项
1. IDA必须是安装版的，我以前用的是免安装版的。
2. python版本、IDA版本，IDAPyhton版本必须匹配。（没有对应python3的版本）
3. python、IDA、IDAPython必须都是32位的或者都是64位的。

# 安装关键文件
1. python27.dll（我安装的是python2.7,如果安装的是pyhton2.6那就是python26.dll）。
	* 这个文件在system32中找（需要考虑重定位到sysWOW64）
2. python.cfg文件。
3. plugins中的python.plw和python.p64。
4. python文件夹里的文件。

# 具体安装步骤
示例IDA版本是6.6，python版本是2.7。
1. 安装Python：Python官网http://www.python.org/getit/。
	* 选择对应操作系统类型及位数。
	* 只能下载python2，因为idapython还没有对应python3的版本
2. 下载相应版本的IDAPython：https://github.com/idapython/bin。
	* IDA版本和Python版本都要和自己机器上安装的版本相对应。
3. 将IDAPython 压缩包解压，得到IDAPython文件夹。
	a. 将解压后的Python文件夹内的所有内容覆盖掉IDA原有Python文件夹（IDA安装目录下）下面的内容。
	b. 将解压后的Plugins文件夹的python.plw和python.p64拷贝到IDA原有Plugins文件夹（自定义，一般IDA安装目录下）下。
	c. 将解压后的python.cfg文件拷贝到IDA原有cfg文件夹（IDA安装目录下）下。
4. 把python27.dll复制到IDA安装目录下。
	* python的系统位数要和IDAPython的系统位数相同。

# 效果
重启IDA后，
* File->Script Command选项中可以选择脚本语言为python
* File->Script files选项中可以选择py文件。
