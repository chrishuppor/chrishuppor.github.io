---
layout: post
title: "MyLittleToolsInstruction"
date: 2019-05-23 8:8:8
categories: Instruction_Tools
tags: MyLittleTools
---
All My Little Tools

所有小工具的源码获取地址： [https://github.com/chrishuppor/src/tree/master/MyLittleTools](https://github.com/chrishuppor/src/tree/master/MyLittleTools)

小工具的可执行程序获取地址：[https://github.com/chrishuppor/src/tree/master/MyLittleTools/bin](https://github.com/chrishuppor/src/tree/master/MyLittleTools/bin)

# FormatErrCode

* 功能：系统错误码描述工具
  * 输入err code回车，程序显示对应英文描述
* 操作
  * 输入GetLastErr()获得的&lt;err code>，回车
  * 输入任意字母结束程序


# DllInjectTool

* 功能：dll注入卸载工具
  * dll注入，支持特定进程注入和全部进程注入
  * 支持dll卸载
* 操作
  * cmd启动，输入相应的启动参数
* 启动参数说明
  * 参数1-可以是pid，可以是进程名。如果使用“*”则尝试注入所有进程
  * 参数2-dll绝对路径（未测试中文路径）
  * 参数3-模式“-i”表示注入，“-d”表示卸载，不区分大小写

# Base64

* 功能：base64编解码
  * 解码base64和字符串base64编码
* 操作
  * 输入要处理的字符串，然后输入0或1。（0表示编码，1表示解码）
* 注意：
  * 这个程序有输入字符的限制
  * 没有对输入数据进行判断，默认输入的都是合法数据

ps:所以说这个程序烂的很，比errcode那个还要充数

# FastDirOpen

* 功能及操作：快速目录启动工具
  * 添加、删除目录——支持多路径拖拽添加，不可以重复添加；右键、Del删除指定路径（仅支持单路径删除）
  * 弹出对应文件夹界面——双击目录、单击+回车、上下键选择+回车
  * 修改备注——两次单击或右键菜单修改备注
  * 排序——单击列标题可按列值从小到大排列
  * 支持无效路径”清理“——将不存在的路径删除
  * 支持全部路径“清空”——删除全部路径
  * 最小化时最小化到系统托盘
  * 启动时保留原来的配置（配置文件为同目录下mydeardirs）
  * 单实例运行，如果已有实例在运行则弹出该实例界面并关闭自己
* 无关功能
  * 列表区域可以跟随界面缩放，不影响美观

PS>点解会有呢个程序嚟？还不是因为windows的快速访问太烂，而我又不想每回都要去一堆文件夹中找目标文件

对这个程序还是比较满意，但仍有两处地方需要处理：

1. 360总是提示该程序在利用文件资源管理器，添加了信任程序还是不行
2. 图标：这图标是模拟人生中我家的一张照片，丑爆了
3. 删除：支持多路径删除

# MD5Calc

* 功能：计算文件或字符串的md5值
* 操作：
  * 输入
    * 字符串：在str文本框输入字符串
    * 文本路径：在file文本框输入文件路径，也可以使用文件浏览，也支持文件拖拽（PS:感觉我像是文件的亲妈字符串的后娘）
  * 计算：点击Calc按键，如果是文件则支持回车进行计算
* （MFC源码丢失，只保存了console版源码）

# spell_with_pic.py

* 功能：获取微信好友头像并拼成指定汉字
* 操作：运行后输入汉字图片路径和头像图片存储文件夹
* 要求：图片存储文件夹中不能有其他文件
* 结果：显示出拼好的图片

# FileTimeChanger

* 功能：批量修改文件时间（包括创建时间、访问时间、更新时间），支持三中时间之间的同步和自定义时间
* 操作：很简单，看了就会
* 目的：有些博客文章希望显示创建时间，有些频繁更新的文章希望显示更新时间，然而已有的修改程序不能满足批量的任意修改文件时间的需求，只好自己动手。

# PrintFileInfoList

* 功能：输出文件夹中文件列表到文件中，同时输出文件的时间信息

# EasyTreeHole

* 功能：简易文本透明加密工具。加密后的文本只能使用本软件和对应的密码打开。