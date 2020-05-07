---
layout: post
title: "python打包成exe"
pubtime: 2019-9-6
updatetime: 2019-9-6
categories: Program
tags: Python Tools
---

python工程打包成exe程序。


# python打包成exe

使用第三方库pyinstaller对python项目进行打包。

* 官方网址：[http://www.pyinstaller.org/](http://www.pyinstaller.org/)

* 官方文档：[https://pyinstaller.readthedocs.io/en/stable/usage.html](https://pyinstaller.readthedocs.io/en/stable/usage.html)

简要说明：

* pyinstaller安装好后会将自己添加到系统环境，可以直接使用。

* 打包好的程序在生成的dist文件夹中，build为临时文件目录。默认路径为当前路径同目录下的dist和build

* 需要全英文路径

* 最简单的命令：pyinstaller  <入口py文件路径\>

资源打包：

pyinstaller在打包py前会自动生成一个spec文件，然后根据这个文件进行打包。开发人员也可以自行编写spec，然后使用```pyinstaller -F <spec文件路径>```打包。

使用spec文件进行打包。spec文件存储了py代码路径和依赖的资源信息，示例如下：

```python
# -*- mode: python ; coding: utf-8 -*-

block_cipher = None

a = Analysis(['C:\\Users\\chris\\Desktop\\new\\setup.py'],
             pathex=['C:\\Users\\chris\\Desktop'],
             binaries=[],
             datas=[],
             hiddenimports=[],
             hookspath=[],
             runtime_hooks=[],
             excludes=[],
             win_no_prefer_redirects=False,
             win_private_assemblies=False,
             cipher=block_cipher,
             noarchive=False)
pyz = PYZ(a.pure, a.zipped_data,
             cipher=block_cipher)
exe = EXE(pyz,
          a.scripts,
          [],
          exclude_binaries=True,
          name='setup',
          debug=False,
          bootloader_ignore_signals=False,
          strip=False,
          upx=True,
          console=False )
coll = COLLECT(exe,
               a.binaries,
               a.zipfiles,
               a.datas,
               strip=False,
               upx=True,
               upx_exclude=[],
               name='setup')
```

其中：

* Analysis:是打包信息的核心内容，包括需要打包的py文件、py文件所在目录；datas是需要引用的文件（图片等）

* exe：name是exe文件的名字， console是是否在打开exe文件时打开命令框（也可以使用--console参数）

* coll：收集前三个部分的内容进行整合