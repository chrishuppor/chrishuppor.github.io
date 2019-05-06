---
layout: post
title: "Reversing.kr_ImagePrc"
date: 2019-5-6 22:42:37
categories: WriteUp
tags: Reversing.kr
---

微简RE challenge网站，my_4ear_3hr1s 就系我啦，6题解题过程及题后思考记录如下。

# Reversing.kr_ImagePrc

## 6.ImagePrc

### 解题过程

1. 拖进PEiD，virustotal没有发现异常。

2. 使用PEView查看IAT发现程序应该是使用WinMain的图形化程序。

3. 运行程序，果然是个画图的程序，点击check后会弹出wrong的提示框。没有可以输入字符串的地方，所以可能是要画什么东西。

4. 使用ResourceHacker查看程序资源，发现程序有一个奇怪的资源——大片的F和某些0组成的二进制数据，很容易使人联想到扫雷的地图和之前编写的迷宫的地图。因此猜想，要画的东西可能与这些0有关，可能是画的东西要落在这些0表示的位置。

5. 拖进IDA

   1. 查找wrong的位置

      wrong的引用出现在sub_401130函数中，查看这个函数的引用，发现这是一个WinProc函数。

   2. 分析WinProc函数

      msg == 0x111表示是系统消息(WM_COMMAND)，此时wparam == 100表示按下了check键

      除了按下check键之后的代码，其他代码都是winmain程序正常运行所需的代码，所以关键就在这里了。代码如下：

      ```c
      if ( wParam == 100 )                        // 按下Check
          {
            GetObjectA(hbm, 24, &pv);
            memset(&bmi, 0, 0x28u);
            bmi.bmiHeader.biHeight = cLines;
            bmi.bmiHeader.biWidth = v16;
            bmi.bmiHeader.biSize = 40;
            bmi.bmiHeader.biPlanes = 1;
            bmi.bmiHeader.biBitCount = 24;
            bmi.bmiHeader.biCompression = 0;
            GetDIBits(hdc, (HBITMAP)hbm, 0, cLines, 0, &bmi, 0);
            v8 = operator new(bmi.bmiHeader.biSizeImage);
            GetDIBits(hdc, (HBITMAP)hbm, 0, cLines, v8, &bmi, 0);
            v9 = FindResourceA(0, (LPCSTR)0x65, (LPCSTR)0x18);
            v10 = LoadResource(0, v9);
            v11 = LockResource(v10);
            v12 = 0;
            v13 = v8;
            v14 = v11 - (_BYTE *)v8;
            while ( *v13 == v13[v14] )
            {
              ++v12;
              ++v13;
              if ( v12 >= 90000 )
              {
                sub_401500(v8);
                return 0;
              }
            }
            MessageBoxA(hWnd, Text, Caption, 0x30u);
            sub_401500(v8);
            return 0;
          }
          return 0;
        }
      ```

      经过查询得知，GetObjectA用于获得一个图形对象，GetDIBits用于获得一个指定位图的位，然后将其作一个DIB使用的指定格式复制到一个缓冲区中，FindResourceA和LoadResource是资源加载时用的函数。while循环中显然是一个数组比较，如果比较未通过则弹出wrong提示框，如果通过则结束。

      至此，程序的逻辑就比较明了了：点击check后将用户画的图与程序自带的资源图相比较，如果比较不过就提示wrong，比较通过的就结束。那么Flag应该就是画出来的字符。

6. 构造正确的图像

   不可能通过画图找到与资源匹配的图像，直接将资源转换为图像就可以了。那么就需要知道图像的一些信息，包括大小、位数、格式。

   1. 通过上面代码中```bmi.bmiHeader.biBitCount = 24```可以知道这是一个24位的图像。

   2. 拖进OD，运行至GetDIBits，查看栈中参数可知这个图像宽200高150，是个RGB图像

      ![Snipaste_2019-05-06_20-17-27.PNG](https://raw.githubusercontent.com/chrishuppor/imgDepot/master/Snipaste_2019-05-06_20-17.PNG)

   3. 将RH中的数据复制到文件中，使用python脚本将十六进制数转变为24位RGB图像，得到的图像上果然有Flag。

      ```python
      from PIL import Image
      
      fr = open('12.txt', 'r')
      text = fr.read()
      fr.close()
      
      rgb_str = ''
      for i in range(0, len(text)):
          if text[i] == 'F' or text[i] == '0':
              rgb_str = rgb_str + text[i]
      
      y = 200
      x = 150
      count = 0
      im = Image.new('RGB', (x, y)) #这里x y如果弄反了也显示不出来
      for i in range(0,x):
          for j in range(0,y):
              r = rgb_str[count] + rgb_str[count + 1]
              g = rgb_str[count + 2] + rgb_str[count + 3]
              b = rgb_str[count + 4] + rgb_str[count + 5]
              im.putpixel((i,j),(int(r, 16),int(g, 16),int(b, 16)))#rgb转化为像素
              count = count + 6
      im.save('1.bmp')
      ```

      *当然可以在网上搜到大佬写的writeup中的简洁代码，这里只贴了拙作*

### 小结

* 使用图像显示出Flag的想法很独特，但是难度不大，通过关键的API也很容易猜出作者意图。解题的关键就在于看出图片比较的代码。
* PIL是python用于处理图像数据的库，需要安装pillow，使用时引入PIL库。