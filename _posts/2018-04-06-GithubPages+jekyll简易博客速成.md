---
layout: post
title: "博客搭建"
date: 2018-04-06 8:8:8
categories: Guide_Installtion
tags: Blog
---
使用GithubPages搭载jekyll创建个人博客（不免费，域名要买的）。

# 为什么这样搭博客

* 开销少  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;Github提供**免费而且稳定的空间**，这样就不必承担空间的费用了。
* 功能够用  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;Github提供**项目主页**的功能，这样就可以利用项目主页撰写博客。  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;Github的项目主页可以进行**域名绑定**，这样就可以将主页与自己的博客域名进行绑定。
* 技术简单  
&#160;&#160;&#160;&#160;&#160;&#160;&#160;使用Github可以不必理会网站的搭建，使用jekyll可以不必理会网页的编写，只需要根据一定格式专心写博文。（*就连我这样不懂web的新手在一天之内就可以搞定。*）

# 配件说明

搭建博客需要三样东西:域名、空间、网页代码
1. GithubPages提供主机空间
2. jekyll用来构建网页代码
3. 域名在网上购买

# 域名购买

&#160;&#160;&#160;&#160;&#160;&#160;&#160;常用的域名购买网站有[GoDaddy](https://sg.godaddy.com/zh)、[Namecheap](https://www.namecheap.com/)、[name](https://www.name.com/zh-cn/)等。

&#160;&#160;&#160;&#160;&#160;&#160;&#160;这一步不存在技术问题，主要是通过查阅资料找到一个**性价比**合适的网站。另外，无论使用哪个域名网站购买，都要注意搜一下**优惠码**，确实可以省一些钱。

&#160;&#160;详情参考:[现在去哪里买 .com 域名最便宜？ - 知乎](
https://www.zhihu.com/question/19551906)\
&#160;&#160;推荐参考:[现在去哪里买 .com 域名最便宜？ - 范进的回答 - 知乎](
https://www.zhihu.com/question/19551906/answer/31986656)

# 创建GithubPages

1. 首先注册一个Github账号
    - eg: xiaoerge
2. 在Github创建一个项目（repository）  
![](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/newrepository.PNG)  
&#160;&#160;项目名称（
Repository name）设为“账号.github.io"。(eg:xiaoerge.github.io)

这样就可以使用“用户名.github.io”访问项目主页了。

# 搭建jekyll环境

在本机搭建jekyll环境的主要目的是为了能够在本地查看博客网页的效果。  
[参考文章: Github+Jekyll —— 创建个人免费博客（二）Ruby+Jekyll部署](https://blog.csdn.net/linshuhe1/article/details/51143274)

## 安装ruby

jekyll是基于ruby的，所以要先安装ruby。
1. 去[Ruby官网](https://rubyinstaller.org/downloads/)下载对应版本的RubyInstallers
    * 要选择WITH DEVKIT的版本（否则要另外安装DEVKIT）
2. 双击下载好的Ruby的安装程序
    * 记好安装目录（之后要用）。
3. 将“安装目录/bin”添加到环境变量的“Path”变量中
4. 在cmd中运行“ruby --version”查看是否安装成功

PS.参考文章中说要安装DEVKIT，但是选择WITH DEVKIT版本的installer的我没有装。

## 安装jekyll
1. 在cmd运行“gem install jekyll”
    * 可能出现“ERROR:Could not find a valid gem "jekyll <>=0> in any repository"”的问题
        * 原因:国外的gems下载地址(https://rubygems.org/)往往很不稳定，需要切换使用中国的gems源下载地址(不只一个，可以多搜几个)
        * 解决办法:
        ```
        gem sources --remove https://rubygems.org/
        gem sources -a https://ruby.taobao.org/
        ```
2. 运行“jekyll --version”检查是否成功

## 其他
1. 安装解析markdown标记的包
    ```
    gem install rdiscount
    ```
另外，如果提示缺少什么，就是用gem安装就好。

如有问题请参考:[在 Windows 上搭建本地 Jekyll 编译环境时问题汇总](https://blog.csdn.net/wudalang_gd/article/details/74619791)

# 绑定域名
本人使用DNSpod进行域名绑定的
1. 在github的xxx.github.io项目根目录下创建文件“CNAME”，文件内容为我们之前购买的域名。这样访问xxx.github.io时就会跳转到购买的域名。
    * 如果没有跳转，可能是浏览器缓存的原因，清空一下缓存（*PS:之后更新博客时也需要清空浏览器缓存，才能看到更新后的博客*）
	* github-pages没有为每一个用户的github-pages分配IP，而是有一个github-pages解析服务器，然后这个服务器会根据域名查找对应github-pages返回。这个域名可以使xxx.github.io，也可以是项目下CNAME中的域名。
2. 在DNSpod进行域名解析设定
    * 添加域名  
    输入购买好的域名
    * 添加记录
        * A记录  
        A记录是将域名解析为IP地址的记录，记录名是IP地址（这里就是github的GithubPages解析服务器的IP地址，就是PING xxx.github.io时返回的IP）
        * CNAME记录  
        CNAME记录是别名记录，记录值是别名，将本域名解析为记录值的域名。（购买好的域名的别名就是xxx.github.io）
3. 设定购买的域名的dns服务器
    * 到购买域名的网站，将其DNS服务器修改为DNSpod的DNS服务器:f1g1ns1.dnspod.net 和 f1g1ns2.dnspod.net
    * DNSpod中有修改dns服务器的教程
4. 等待...国际域名服务器更新大概要等待72小时，所以72小时后再访问购买的域名，看看是否被解析到了xxx.github.io的地址。

***
至此，该进行的基础配置已经完成，接下来给出一个最简单的博客
***

# 创建第一个极简博客
1. 在cmd中进入自己喜欢的目录下，运行“jekyll new yyy”
    * 这个yyy文件夹用来在本地存放博客全部内容
    * 也可以是其他名字
2. 在yyy目录下运行“jekyll serve”就可以在本地查看博客效果（地址:127.0.0.1:4000）
3. 将github上的项目xxx.github.io清空（留着CNAME），将yyy文件夹内的全部内容上传到该项目
4. 访问xxx.github.io即可看到博客内容(此时需要清空CNAME,清空浏览器缓存才能访问成功)
***
当然，这个极简博客很丑陋，之后需要进行美化...
* [jekyll theme](http://jekyllthemes.org/)提供了很多博客模板，模板中也提供了使用说明
* [jekyll 教程](http://ju.outofmemory.cn/entry/98471)

# 其他参考文章
1. [GithubPages教程 在GithubPages上搭建项目主页](https://blog.csdn.net/yanzhenjie1003/article/details/51703374)（讲述如何搭建项目主页）
2. [Github+Jekyll —— 创建个人免费博客（四）jekyll第一个页面](https://blog.csdn.net/linshuhe1/article/details/51148645)（环境搭好创建第一个博客，不必纠结为什么有些目录没有）
3. [使用Github Pages建独立博客](http://beiyuu.com/github-pages)（讲述了一整套流程）
4. [通过GitHub Pages建立个人站点（详细步骤）](https://www.cnblogs.com/purediy/archive/2013/03/07/2948892.html)（详细讲述了搭建流程）
5. [使用Jekyll在Github上搭建个人博客（文章分类索引）](https://segmentfault.com/a/1190000000406017)（美化博客的一步——添加文章分类索引）
6. [如何使用 Jacman 主题](http://simpleyyt.com/jekyll-jacman/jekyll/2015/09/20/how-to-use-jacman)（如何套用Jacman模板）