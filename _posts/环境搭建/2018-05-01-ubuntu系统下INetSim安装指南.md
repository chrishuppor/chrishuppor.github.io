---
layout: post
title: "ubuntu系统下INetSim安装指南"
pubtime: 2018-05-01
updatetime: 2018-05-01
categories: EnvironmentBuild
tags: INetSim
---

ubuntu系统下INetSim安装指南。

# ubuntu系统下INetSim安装指南

1. 首先安装一个ubuntu系统，我的是版本16.04。
2. 打开终端，切换到root账户
   1. 输入```su root```
   2. 如果没有设定过root密码则需要先设置root密码——输入```sudo passwd root```，设置root密码
3. 按照[INetSim官方文档](https://www.inetsim.org/packages.html)，依次以下输入指令
   1. ```echo "deb http://www.inetsim.org/debian/ binary/" > /etc/apt/sources.list.d/inetsim.list```
   2. ```wget -O - http://www.inetsim.org/inetsim-archive-signing-key.asc | apt-key add -```
   3. ```apt update```
   4. ```apt install inetsim```

PS：安装INetSim似乎需要运气，我在一年前尝试过几次都安不好，这次却十分顺利。尽管这一年来我确实有很大的进步，但并没有往这个方向发展呀，大概我真的是悟性高且擅长将各种技能的进步内化成核心能力的提升。不开玩笑的，我猜测大概是两个原因：

* 之前安装使用的普通用户，使用sudo命令来安装；这次使用root账户安装。看来root和sudo还是有区别的。
* 在我安装失败之后不久，它更新了。。。