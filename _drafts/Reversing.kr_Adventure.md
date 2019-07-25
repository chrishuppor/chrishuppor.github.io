Reversing.kr中的Adventure，怕是这里面最硬核的题了，因为最少人做出来。不过也很可能是MetroApp的问题。

# 解题过程

1. 安装时需要修改系统时间到证书的有效时间内，安装程序前需要先安装dependencies里面对应的库。

2. plm程序调试可以使用plmdebug.exe。这个是微软提供的调试工具，专门用于调试appx文件，可以通过winSDK获得。

3. 使用命令行将程序与调试器绑定。

   ```plmdebug.exe /enableDebug <package> <调试器路径>```

   其中，package是plm程序的包名，一般与程序名称不同，可以通过```plmdebug /query```查询。这里我使用的是一个叫x32dbg的调试器，因为OD调试时出现了不可解决的异常。

4. 运行程序，查看字符串，发现了两个明显的字符串“Flag1”和“Flag2”。追踪到对应位置，然后根据offset在IDA中查看反编译源码。

   这两个字符串出现在sub_403890中。

5. 本来以为可以修改怪物释放时间，但是发现不可以粗暴的修改4043a0函数，因为之后的字符串异或需要用到rand

# 小结

没有

# 参考文章

* [错误代码0x800b0101解决办法](http://www.uqidong.com/wtjd/2767.html)