Reversing.kr中的Adventure，怕是这里面最硬核的题了，因为最少人做出来。不过也很可能是MetroApp的问题。

# 解题过程

1. 安装时需要修改系统时间到证书的有效时间内，安装程序前需要先安装dependencies里面对应的库。

2. plm程序调试可以使用plmdebug.exe。这个是微软提供的调试工具，专门用于调试appx文件，可以通过winSDK获得。

3. 使用命令行将程序与调试器绑定。

   ```plmdebug.exe /enableDebug <package> <调试器路径>```

   其中，package是plm程序的包名，一般与程序名称不同，可以通过

4. 运行程序

5. 本来以为可以修改怪物释放时间，但是发现不可以粗暴的修改4043a0函数，因为之后的字符串异或需要用到rand

# 小结

没有

# 参考文章

* [错误代码0x800b0101解决办法](http://www.uqidong.com/wtjd/2767.html)