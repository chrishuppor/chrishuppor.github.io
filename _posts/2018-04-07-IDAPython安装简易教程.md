---
layout: post
date:   2018-04-07 15:27:07 +0800
categories: �̳� �����
---

�����·�ԭ������ԭ�Ļ����Ͻ��е��޸ġ�[ԭ�ĵ�ַ](http://www.cnblogs.com/blacksunny/p/7215247.html)

--------------------------------------------------------------------------------

# ע������
  1. IDA�����ǰ�װ��ģ�����ǰ�õ����ⰲװ��ġ�
  2. python�汾��IDA�汾��IDAPyhton�汾����ƥ�䡣��û�ж�Ӧpython3�İ汾��
  3. python��IDA��IDAPython���붼��32λ�Ļ��߶���64λ�ġ�

# ��װ�ؼ��ļ�
  1. python27.dll���Ұ�װ����python2.7,�����װ����pyhton2.6�Ǿ���python26.dll����
  �� ����ļ���system32���ң���Ҫ�����ض�λ��sysWOW64��
  2. python.cfg�ļ���
  3. plugins�е�python.plw��python.p64��
  4. python�ļ�������ļ���

# ���尲װ����
ʾ��IDA�汾��6.6��python�汾��2.7��
  1. ��װPython��Python����http://www.python.org/getit/��
	* ѡ���Ӧ����ϵͳ���ͼ�λ����
	* ֻ������python2����Ϊidapython��û�ж�Ӧpython3�İ汾
  2. ������Ӧ�汾��IDAPython��https://github.com/idapython/bin��
	* IDA�汾��Python�汾��Ҫ���Լ������ϰ�װ�İ汾���Ӧ��
  3. ��IDAPython ѹ������ѹ���õ�IDAPython�ļ��С�
	1. ����ѹ���Python�ļ����ڵ��������ݸ��ǵ�IDAԭ��Python�ļ��У�IDA��װĿ¼�£���������ݡ�
	2. ����ѹ���Plugins�ļ��е�python.plw��python.p64������IDAԭ��Plugins�ļ��У��Զ��壬һ��IDA��װĿ¼�£��¡�
	3. ����ѹ���python.cfg�ļ�������IDAԭ��cfg�ļ��У�IDA��װĿ¼�£��¡�
  4. ��python27.dll���Ƶ�IDA��װĿ¼�¡�
	* python��ϵͳλ��Ҫ��IDAPython��ϵͳλ����ͬ��

# Ч��
����IDA��
* File->Script Commandѡ���п���ѡ��ű�����Ϊpython
*  File->Script filesѡ���п���ѡ��py�ļ���
 