---
title: "python tips"
author: 一张狗
lastmod: 2019-07-06 10:46:42
date: 2018-02-27 11:29:13
tags: []
---


1.字符串当变量名

exec(‘abc = 5’)  
 globals()[‘abc’] = 6  
 setattr(__builtins__, ‘abc’, 9)  
 __import__(‘sys’)._getframe(0)<wbr></wbr>.f_globals[‘abc’] = 27  
 四种都可以实现，那么，对于引用a如何得到{“c”:1}，<wbr></wbr>则应该是：  
 ```
 >>> a=’bbb’  
 >>> bbb={“c”:1}  
 >>> exec(‘a=%s’ % a)  
 >>> a  
 {“c”: 1}
```
参考https://groups.google.com/forum/#!topic/python-cn/njvNp-1Fhrc

 

2.not all arguments converted during string formatting python

打印的时候%不对或者没有%


