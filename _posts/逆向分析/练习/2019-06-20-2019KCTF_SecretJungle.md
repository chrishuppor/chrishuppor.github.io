---
layout: post
title: "2019KCTF_SecretJungle"
date: 2019-06-20 8:8:8
categories: Reverse
tags: WriteUp
---

2019kctf的第五题，一个包了三层技术简单思路巧妙的android逆向题。


# 解题过程

1. 将apk拖入jadx，查看MainActivity。

   MainActivity十分简单，只有一个onCreate方法，该方法主要做了两件事——loadUrl(this.u)和响应按钮点击。

   注意到System.loadLibrary("gogogo")，说明本程序使用了JNI，对应的so库需要解压apk然后在lib文件夹中找到。

   用到了两个外部函数gogogoJNI.sayHello和gogogoJNI.check_key。很明显，关键就在这里，因此需要进一步查看so库。

   ```java
   public class MainActivity extends AppCompatActivity {
       private Button button1;
       private EditText eText1;
       private TextView txView1;
       public String u = gogogoJNI.sayHello();
   
       static {
           System.loadLibrary("gogogo");
       }
   
       /* Access modifiers changed, original: protected */
       public void onCreate(Bundle bundle) {
           super.onCreate(bundle);
           setContentView((int) R.layout.activity_main);
           this.eText1 = (EditText) findViewById(R.id.editText);
           this.txView1 = (TextView) findViewById(R.id.textView);
           ((WebView) findViewById(R.id.text1View)).loadUrl(this.u);
           ((WebView) findViewById(R.id.text1View)).getSettings().setJavaScriptEnabled(true);
           this.button1 = (Button) findViewById(R.id.button);
           this.button1.setOnClickListener(new OnClickListener() {
               public void onClick(View view) {
                   if (gogogoJNI.check_key(MainActivity.this.eText1.getText().toString()) == 1) {
                       MainActivity.this.txView1.setText("Congratulations!");
                   } else {
                       MainActivity.this.txView1.setText("Not Correct!");
                   }
               }
           });
       }
   }
   ```

2. 本程序使用JNI引入了c生成的so库，该库名称为libgogogo.so。分析x86平台的libgogogo.so如下。

   1. 拖入ida，很容易就看到了两个关键函数Java_com_example_assemgogogo_gogogoJNI_check_key和Java_com_example_assemgogogo_gogogoJNI_sayHello。

   2. 查看函数时，发现两个函数都通过第一个参数a1动态调用了函数。经查询，该参数是java通过JNI调用c代码编写的库时传递给接口函数的JNIEnv结构体，该结构体定义在jni.h中。经查询，可知，x86版libgogogo.so中动态调用的JNIEnv中函数如下：

      a1 + 676：JNIEnv中第（676/4 + 1）个函数——GetStringUTFChars

      a1 + 680：ReleaseStringUTFChars

      a1 + 668：NewStringUTF

   3. 查看Java_com_example_assemgogogo_gogogoJNI_check_key函数。函数十分简单，就是用一个while循环遍历判断输入的字符串，如果都通过则返回1，但其循环条件十分古怪：```*(_BYTE *)(v4 + v6) == (rand() % 128 != *v5)```，其中*v5是输入的字符，无论如何也不会是1或0，也就是说这个check永远不会通过，说明真正的check不在这里。

      通过后续分析可知，这个函数只是用来迷惑破解者的。

   4. 查看Java_com_example_assemgogogo_gogogoJNI_sayHello函数，发现该函数计算了一个url地址并返回。该地址为127.0.0.1:8000。而apk的MainActivity中使用loadUrl加载了该地址。此时，猜测可能通过自己跟自己网络通信传递了其他数据进行check。

3. 经查询可知，调用JNI会在loadlibrary时调用JNI_OnLoad函数。因此接下来重点查看JNI_OnLoad函数。

   1. 从JNI_OnLoad逐层深入，发现了inti_proc函数，该函数主要有以下行为：

      1. 创建一个socket并绑定了8000端口，并且进行监听。
      2. 以sub_A30函数为线程函数，以服务端的socket为参数，创建了6个线程。
      3. 对byte_3004的数据进行了异或解密

      解密了byte_3004，却没有在本函数用到，需要重点关注。

   2. 查看sub_A30函数，该函数负责接受来自客户端的socket请求，并send一组数据给客户端，这组数据正好就是byte_3004。

   3. 使用idapython脚本获取byte_3004的数据并解密，print解密结果后发现这是一个html文件。

      脚本代码如下：

      ```python
      import idaapi
      
      ea = 0x3004
      x = []
      for i in range(0, 34291):
          eb = ea + i
          x.append(Byte(eb))

      for i in range(0, 34291):
          x[i] = x[i] ^ 0x67
      
      t = ''
      for i in range(0, 34291):
          t += chr(x[i])
      
      print(t)
      fw = open("hide.html", "w")
      fw.write(t)
      fw.close()
      ```

4. 将获得的html文件运行一下，发现界面与apk运行界面一模一样，至此已经了解作者的思路——首先自身启动一个127.0.0.1的socket服务器并准备好html数据，然后使用loadUrl打开html，接收用户key并check。启动的功能在JNI_Onload函数中完成，连接的功能在MainActivity中完成。

   至于apk中的按键处理实则与整个程序无关，如果细看的话可能apk中根本就没有整个按钮资源。（后来猜的，没有验证）

5. 查看html代码，关键如下

   ```javascript
     var ret =  instance.exports.check_key();
   
     if (ret == 1){
      document.getElementById("tips").innerHTML = "Congratulations!"
     }
     else{
       document.getElementById("tips").innerHTML = "Not Correct!"
     }
   ```

   其中instance.exports.check_key()是通过WebAssembly.compile根据二进制代码动态获得的函数，因此这里的大片二进制数据即为一个WASM文件。

   所以html中的check是通过wasm中的代码来完成的。

6. 将html中的wasm文件提取出来（即check.wasm文件），使用wasm2c反编译为c代码（即check.c），再使用gcc编译check.c获得中间文件check.o。

7. 将check.o拖入ida，在函数列表中就可以找到check_key函数，反编译结果如下

   ```c
   int check_key()
   {
     int v0; // eax
     int result; // eax
     int v2; // [esp-Ch] [ebp-24h]
     int v3; // [esp-8h] [ebp-20h]
     int v4; // [esp-4h] [ebp-1Ch]
   
     if ( ++wasm_rt_call_stack_depth > 0x1F4u )
       wasm_rt_trap(7, v2, v3, v4);
     o(1024, 1025, 1026, 1027);
     oo(1028, 1029, 1030, 1031);
     ooo(1032, 1033, 1034, 1035);
     oooo(1036, 1037, 1038, 1039);
     ooooo(1040, 1041, 1042, 1043);
     oooooo(1044, 1045, 1046, 1047);
     ooooooo(1048, 1049, 1050, 1051);
     v0 = oooooooo(1052, 1053, 1054, 1055);
     result = xxx(v0, 1053, 1054, 1055);
     --wasm_rt_call_stack_depth;
     return result;
   }
   ```

8. 查看o函数，发现其关键代码如下：

   ```c
   i32_store8(&memory, a1, 0, v25 ^ 0x18);
   v6 = i32_load8_u(&memory, a2, 0);
   i32_store8(&memory, a2, 0, v6 ^ 9);
   v7 = i32_load8_u(&memory, a3, 0);
   i32_store8(&memory, a3, 0, v7 ^ 3);
   v8 = i32_load8_u(&memory, a4, 0);
   i32_store8(&memory, a4, 0, v8 ^ 0x6B);
   v22 = i32_load8_s(&memory, a1, 0) ^ 0x70;
   ```

   也就是o函数负责对memory中第1024, 1025, 1026, 1027的数据进行异或。

   其余oo、ooo等函数也是这个功能。

9. 查看xxx函数，该函数对memory中1024-1055的数据进行了线性计算，如果通过计算则check通过。因此要获得flag只需要解一个线性方程组，然后异或一下。

10. 编写脚本获得flag:

   其中，xorarg为异或的数组，famula.txt为xxx中的方程字符串。

   ```python
   import numpy as np
   from scipy.linalg import solve
   
   def generate2dimarray(m, n):
   	ret = []
   	for i in range(m):
   		item = []
   		for j in range(n):
   			item.append(0)
   		ret.append(item)
   	return ret
   
   xorarg = [0x18,9,3,0x6b,1,0x5a,0x32,0x57,0x30,0x5d,0x40,0x46,0x2b,0x46,0x56,0x3d,2,0x43,0x17,0,
   0x32,0x53,0x1f,0x26,0x2a,1,0,0x10,0x10,0x1e,0x40,0]
   
   fr = open("famula.txt", "r")
   famulalines = fr.readlines()
   fr.close()
   
   a_array = generate2dimarray(32, 32)
   b_array = [0,0,0,0,0,0,0,0,0,0,
   			0,0,0,0,0,0,0,0,0,0,
   			0,0,0,0,0,0,0,0,0,0,
   			0,0]
   linecount  = 0
   for fline in famulalines:
   	if "&&" in fline:
   		linecount += 1
   		bpos = fline.index("&& ") + 3
   		epos = fline.index(" * ")
   		atmp = int(fline[bpos: epos])
   
   		xpos = fline.index("n") + 1
   		xtmp = int(fline[xpos: xpos + 4]) - 1024
   		a_array[linecount][xtmp] = atmp
   		#print (atmp, xtmp)
   	else:
   		bpos = fline.index("+ ") + 2
   		epos = fline.index(" * ")
   		atmp = int(fline[bpos: epos])
   
   		xpos = fline.index("n") + 1
   		xtmp = int(fline[xpos: xpos + 4]) - 1024
   		a_array[linecount][xtmp] = atmp
   		#print (atmp, xtmp)
   
   	if "==" in fline:
   		bpos = fline.index("== ") + 3
   		btmp = int(fline[bpos: bpos + 6])
   		b_array[linecount] = btmp
   
   aa = np.array(a_array)
   bb = np.array(b_array)
   x = solve(aa, bb)
   
   flag = ''
   for i in range(32):
   	tmp = int(round(x[i])) ^ xorarg[i]
   	flag += (chr(tmp))
   print(flag) #K9nXu3_2o1q2_w3bassembly_r3vers3
   ```

# 小结

* 本程序使用JNI调用了c编写的so库。
  * JNI是在java中调用c代码的一种方法，有约定好的数据结构和函数样式，数据结构定义在jni.h文件中。
  * 这个so库在解压apk后的lib文件夹中，并且有不同的版本。
* 本程序通过socket来在函数之间传递数据，方法奇妙。*（PS：我猜也可以抓包获得字符串，未验证。）*
* 本题在js中通过webassembly解析wasm二进制代码来动态调用函数，这是在js中调用c代码的一种方式。wasm二进制代码是由c源码逐步生成，可以通过wasm2c工具反编译为c文件。但是这个c文件是很难看的，可以使用gcc再编译获得o文件，使用IDA分析o文件。

#  总结

* 逆向的知识千千万，学是不可能学完的，关键要学会搜索。

# 参考博客

* [TSCTF Android Reverse - Open Sesame!](https://thinkycx.me/posts/2019-05-13-TSCTF-Android-Reverse-Open-Sesame!.html)
* [android jni.h源码](https://blog.csdn.net/kevinffk/article/details/84740699)
* [一种Wasm逆向静态分析方法](https://xz.aliyun.com/t/5170)
* [使用wasm2c反编译wasm代码](https://blog.csdn.net/weixin_33748818/article/details/88110604) (ps. wabt的编译最好参考该项目中的readme文件)

# 附录

解题过程中生成的中间文件:[SecretJungle解题中间文件.zip](https://github.com/chrishuppor/attachToBlog/tree/master/SecretJungle解题中间文件.zip)