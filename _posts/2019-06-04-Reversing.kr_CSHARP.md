---
layout: post
title: "Reversing.kr_CSHARP"
date: 2019-6-4 8:46:27
categories: WriteUp
tags: Reversing_kr
---


Reversing.kr的CSHARP，是一个动态解密函数的程序。


# 解题过程

1. 拖入PEiD得知这是一个C#程序。

2. 拖入dnSpy，查看入口函数。

   这个函数十分简单，就是创建了一个Form1的对象。

   ```c#
   private static void Main()
   {
       Application.EnableVisualStyles();
       Application.SetCompatibleTextRenderingDefault(false);
       Application.Run(new Form1());
   }
   ```

3. 查看Form1构造函数

   如下，首先获取了MetMett函数的二进制代码并赋值给了Form1.bb，然后对Form1.bb进行了一些变换，可能是对MetMett代码的解密操作。

   ```c#
   public Form1()
   {
       Form1.bb = base.GetType().GetMethod("MetMett", BindingFlags.Static | BindingFlags.NonPublic).GetMethodBody().GetILAsByteArray();
       byte b = 0;
       for (int i = 0; i < Form1.bb.Length; i++)
       {
           byte[] array = Form1.bb;
           int num = i;
           array[num] += 1;
           b += Form1.bb[i];
       }
       Form1.bb[18] = b - 38;
       Form1.bb[35] = b - 3;
       Form1.bb[52] = (b ^ 39);
       Form1.bb[69] = b - 21;
       Form1.bb[87] = 71 - b;
       Form1.bb[124] = (b ^ 114);
       Form1.bb[141] = (b ^ 80);
       Form1.bb[159] = 235 - b;
       Form1.bb[179] = 106 + b;
       Form1.bb[200] = 36 - b;
       Form1.bb[220] = b - 3;
       this.InitializeComponent();
   }
   ```

4. 查看MetMett，果然这个函数代码有问题，所以 Form1()就是对这段代码进行了解密。但是 Form1()并没有将解密结果写回这个函数，需要关注一下Form1.bb的使用。

5. 查看btnCheck_Click函数，发现它只调用了MetMetMet函数，并且以用户输入为参数。说明MetMetMet函数就是匹配的核心函数。

   ```c#
   Form1.MetMetMet(this.txtAnswer.Text);
   ```

6. 查看MetMetMet函数

   1. 这个函数首先将用户输入转为base64编码字符串。

      ```c#
      byte[] bytes = Encoding.ASCII.GetBytes(Convert.ToBase64String(Encoding.ASCII.GetBytes(sss)));
      ```

   2. 然后使用TypeBuilder和MethodBuilder创建了一个名为MetM的动态方法，这个方法的代码就是Form1.bb。

      ```c#
      TypeBuilder typeBuilder = AppDomain.CurrentDomain.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.RunAndSave).DefineDynamicModule(assemblyName.Name, assemblyName.Name + ".exe").DefineType("RevKrT1", TypeAttributes.Public);
      MethodBuilder methodBuilder = typeBuilder.DefineMethod("MetMet", MethodAttributes.Private | MethodAttributes.Static, CallingConventions.Standard, null, null);
      TypeBuilder typeBuilder2 = AppDomain.CurrentDomain.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.RunAndSave).DefineDynamicModule(assemblyName.Name, assemblyName.Name + ".exe").DefineType("RevKrT2", TypeAttributes.Public);
      //here
      typeBuilder2.DefineMethod("MetM", MethodAttributes.Private | MethodAttributes.Static, CallingConventions.Standard, null, new Type[]
                                    {
                                        typeof(byte[]),
                                        typeof(byte[])
                                    }).CreateMethodBody(Form1.bb, Form1.bb.Length); 
      Type type = typeBuilder2.CreateType();
      MethodInfo method = type.GetMethod("MetM", BindingFlags.Static | BindingFlags.NonPublic);
      ```

   3. 然后以[1,2]和base64编码的输入为参数调用了MetM函数

      ```c#
      method.Invoke(obj, new object[]{array, bytes});
      ```

   4. 最后根据array[0]的值判断匹配是否通过。由此可见，匹配的核心代码就是Form1.bb中的数据。直接看二进制数据实在太难了，最好是有反编译源码——只要将Form1.bb中的数据替换掉文件中的MetMett函数的数据就可以在dnSpy看到反编译源码了。

7. 在Form1构造函数下断点，解密完成后，利用dnSpy从内存中获取Form1.bb的数据。这个数据就是MetMett函数正确的二进制数据。

8. 在exe文件中定位MetMett函数位置。这个位置不是dnSpy显示的0x52C，而需要根据MetMett函数二进制数据自己查找。然后将该函数代码替换为正确的代码。

   替换脚本：

   ```python
   f1 = open("CSharp.exe", 'rb')
   b1 = f1.read();
   f1.close()
   
   fw = open("changed.exe", "wb")
   
   fr = open("MetMett.txt", "r")
   text = fr.read()
   fr.close()
   
   a = []
   tmp = 0
   for i in range(0,len(text)):
       if i % 2 == 1:
           tmp = tmp * 16 + int(text[i], 16)
           a.append(tmp)
           else:
               tmp = int(text[i], 16)
   
               b = bytearray(b1)
               for i in range(0x538, 0x538 + 234):
                   b[i] = a[i - 0x538]
                   
   fw.write(b)
   fw.close()
   ```

9. dnSpy打开解密后的exe文件，MetMett函数反编译代码如下，其中bt为输入字符串的base64编码数据，chk是数组[1,2]。

   ```c#
   private static void MetMett(byte[] chk, byte[] bt)
   {
       if (bt.Length == 12)
       {
           chk[0] = 2;
           if ((bt[0] ^ 16) != 74)
           {
               chk[0] = 1;
           }
           if ((bt[3] ^ 51) != 70)
           {
               chk[0] = 1;
           }
           if ((bt[1] ^ 17) != 87)
           {
               chk[0] = 1;
           }
           if ((bt[2] ^ 33) != 77)
           {
               chk[0] = 1;
           }
           if ((bt[11] ^ 17) != 44)
           {
               chk[0] = 1;
           }
           if ((bt[8] ^ 144) != 241)
           {
               chk[0] = 1;
           }
           if ((bt[4] ^ 68) != 29)
           {
               chk[0] = 1;
           }
           if ((bt[5] ^ 102) != 49)
           {
               chk[0] = 1;
           }
           if ((bt[9] ^ 181) != 226)
           {
               chk[0] = 1;
           }
           if ((bt[7] ^ 160) != 238)
           {
               chk[0] = 1;
           }
           if ((bt[10] ^ 238) != 163)
           {
               chk[0] = 1;
           }
           if ((bt[6] ^ 51) != 117)
           {
               chk[0] = 1;
           }
       }
   }
   ```

   匹配逻辑十分简单，解密脚本如下：

   ```python
   b = [74, 70, 87, 77, 44, 241, 29, 49, 226, 238, 163, 117]
   x = [16, 51, 17, 33, 17, 144, 68, 102, 181, 160, 238, 51]
   pos = [0, 3, 1, 2, 11, 8, 4, 5, 9, 7, 10, 6]
   
   res = [0 for i in range(12)]
   for i in range(12):
       res[pos[i]] = b[i] ^ x[i]
   
   resstr = ''
   for i in range(12):
       resstr += chr(res[i])
   print (resstr)
   ```

# 小结

* MetMett函数二进制代码解密算法十分简单，因此也可以自己脚本解密。
* MetMett函数在exe文件中的地址要自己查，dnSpy给出的不靠谱。