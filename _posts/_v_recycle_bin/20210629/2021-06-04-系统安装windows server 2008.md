---
layout: post
title: "系统安装windows server 2008"
pubtime: 2021-06-04
updatetime: 2021-06-04
categories:  EnvironmentBuild
tags: windows
---

使用GHO镜像、驱动总裁安装windows server 2008 r2系统。由于某些桌面主板自带的Intel网卡没有server版本的驱动，还需要手动安装网卡驱动。

# 前言

在近几年新出的主板上安装windows server 2008 r2和安装windows 7一样都会遇到驱动的问题，原因就是主板和系统都没有提供USB驱动，需要手动安装。

由于Windows系统原因，一般桌面主板自带的Intel网卡（典型的包括I211、I211-AT、I217-V、I218-V、I219-V）等，都无法在Windows Server系统上找到对应的驱动。但是，这些网卡几乎都有对应的服务器主板版本（例如I219-LM）。这些网卡其实并没有本质上的差别，只是在驱动层面，利用不同的驱动签名使得网卡不能通用。(摘自：[Windows Server 2019安装Intel I219-V I211网卡驱动](https://blog.csdn.net/bobytomm/article/details/103582264))

# 1.制作windows server 2008 r2的GHO文件和PE盘

制作方法参考B站视频[【使用VMare虚拟机和PE工具自制纯净的GHOST文件】](https://www.bilibili.com/video/BV1Mf4y1a7Nq/)

# 2.准备文件

1. 驱动总裁win7离线版： https://www.sysceo.com/Software-softwarei-id-262.html
2. 下载网卡对应的驱动：支持Intel i211-AT+windows server 2008 r2的网卡驱动https://downloadcenter.intel.com/download/29572/Intel-Ethernet-Adapter-Complete-Driver-Pack

# 3.准备工具

PS2接口的键盘：因为server版的系统需要输入密码才能进入系统，进入系统后才能安装USB驱动，所以密码只能通过PS2接口的键盘来输入。

# 4.安装系统

1. 把PE盘插入机器，开机进入Bios，选择从PE盘引导
2. 进入PE系统后，双击打开DG工具，初始化机器硬盘分区
3. 双击打开Ghost备份还原工具，选择机器硬盘所在分区，选择准备好的GHO文件，选择恢复分区
4. 双击打开驱动总裁文件，双击exe程序，选择系统盘符(就是我们恢复的那个分区所用盘符)，选择“加载驱动到目标系统”
5. 将键盘接到机器上
6. 重启机器

# 5.安装网卡驱动

1. 修改网卡驱动安装信息
    1. 进入“PRO1000”》“Win64”(跟机器位数相关)》“NDIS62”(跟server版本有关，2012选择NDIS64)，用记事本打开文件“elr52*64”
    2. 搜索i211
    ![](_v_images/20210604152225630_8232.png =960x)
    3. 搜索到i211所在选项卡，复制下面的E1539.6.1.1
    ![7](_v_images/20210604152307803_12994.png =960x)
    4. 回到文件顶部，搜索E1539.6.1.1。复制搜索到的第一行，这一行就是安装信息。
    ![9](_v_images/20210604152416445_6818.png =960x)
    5. 将这一行粘贴到下面64.6.1的选项卡中，保存文件。
    ![10](_v_images/20210604152500050_30699.png =960x)
2. 修改系统模式
    管理员权限执行如下命令来关闭签名校验，开启测试模式，否则会安装失败。
    ```
    bcdedit -set loadoptions DISABLE_INTEGRITY_CHECKS
    bcdedit -set TESTSIGNING ON
    ```
    重启机器，进入测试模式。
    PS:安装完成后可用以下命令恢复：
    ```
    bcdedit -set loadoptions ENABLE_INTEGRITY_CHECKS
    bcdedit -set TESTSIGNING OFF
    ```
3. 从磁盘安装驱动
    ![](_v_images/20210604152749085_1407.png =960x)
    ![](_v_images/20210604152818308_32746.png =960x)
    ![](_v_images/20210604152831402_11494.png =960x)
    ![](_v_images/20210604152842696_37.png =960x)
    ![](_v_images/20210604152905065_11007.png =960x)
    ![](_v_images/20210604152922221_25374.png =960x)
    ![](_v_images/20210604152935369_23919.png =960x)
    ![](_v_images/20210604152948694_24675.png =960x)
    ![](_v_images/20210604153004633_13314.png =960x)
    ![](_v_images/20210604153016814_19747.png =960x)
    ![](_v_images/20210604153029385_2424.png =960x)