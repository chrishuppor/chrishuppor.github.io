---
layout: post
title: "windows��pintools�ı�����ʹ��"
pubtime: 2019-09-20
updatetime: 2020-09-20
categories: Reverse
tags: tools pin
---

��¼pin�İ�װ��pintools�ı�����ʹ�á�

# pin��װ

��pin�Ĺ���[http://pintool.org/ ](http://pintool.org/)�������°汾��pin���߰���ע��Ҫ���Լ�������ϵͳƥ�䡣

���������ľ���pin��exe������Ҫ���룬����ֱ��ʹ�ã���Ҫ�Լ��������pintools��

# pintools�ı�����ʹ��

pintools���ʾ���һ����̬���ӿ⡣pintools�ı��������c/c++�ı��빤�߽�toolsԴ��������ӳ�dll(windows��)��

## ʾ��

��pin�Դ���ʾ��inscount0Ϊ�������б�����ʹ�á�(·����"%Pin��װĿ¼%\source\tools\ManualExamples")

����˵Ҫ�Ȱ�װcygwin�����ҵĻ����ﱾ����װ��cygwin�����ұ���pintoolsʱҲû���õ�������Ҳ��֪���ǲ������Ҫװ��

**����**

1. ��vs�ʺϲ���ϵͳ�汾��**������**������"%Pin��װĿ¼%\source\tools\ManualExamples"
2. ִ��`make obj-intel64/inscount0.dll`�������"%Pin��װĿ¼%\source\tools\ManualExamples\obj-intel64"������inscount0.dll�������������Ҫ�õ�pintools.

**ʹ��**

ִ��```<pin.exe��·��> -t <pintools��dll·��> -- <�������ĳ����·��>```
��Ҫע����ǣ�dll�뱻�����ĳ����λ��Ҫһ�¡�

---

���⻰��linux��pintools��ע�����

linux��pin��װҪ��Ӧ�ð汾��gcc�汾��ϵͳλ�����������ڱ���ʱ����һ�Ѷѵ�err���������ҵĸ��˾��飺

* ��ubuntu 18.04 64λ�����ϰ�װpin-3.7�����Ա���Ŀ��ƽ̨Ϊintel64�ĳ��򣬲����Ա���ia32�ĳ���
* ��ubuntu 16.04 32λ�����ϰ�װpin-2.7����Ϊgcc�汾��ƥ�䱨����װpin-3.7��ʮ�����ʡ�