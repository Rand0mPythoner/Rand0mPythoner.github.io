---
layout: post
title: "php代码审计杂谈"
subtitle: "php mother fucker"
author: "P1an0"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - php
  - 代码审计
---

> 2020年搭建了自己的Blog，先写点东西交差吧2333

关于php白盒中常见的Tricks已经很多了，这里就是说一下我常常注意的点。

本文以漏洞作为划分维度。

0x01 Sqli:
========
​	在PHP中，PDO的出现基本上杜绝了字符型Sql注入，但是不需要闭合单引号的Sql注入却被忽略了，我们常常可以在整数型Sqli中直接注入。

​	在Mysql中，如果开发时使用in函数来拼接可以绕过PDO，这里不多赘述了。

0x02 文件包含:
========

​	php文件包含的函数有4个，分别是：
> include
> include_once
> require
> require_once

​	include和require的区别很简单，include是代码执行到该语句时加载，require在php伪编译时便加载了。

​	X_once与X的区别就是如果x_once先包含了一个文件，include再包含该文件会失败，其中include会报错并中断。

​	文件包含支持传入文件流与伪协议的，在我们渗透测试时常用的伪协议有：  
>filter、data、input 、file等

其中filter，file可以用来读取文件，data和input可以直接传入文件流被include执行
使用方法都在百度(逃

**注：  
>data伪协议需要php环境开启allow_url_fopen和allow_url_include
>input伪协议需要php环境开启allow_url_include
>data伪协议在php7中可以使用base64编码注释掉后面的字符串来规避后面的字符串，也就是如果reuqire("{用户可控}/asdasd/asdads.php")也是可以通过base64单行注释来进行命令执行。

0x03 SSRF
====
php中的ssrf Bypass和常规的ssrf一样，直接把trick列出来8  

>前置域名：使用@符号闭合为登陆凭证绕过
>[^@符号之前不能有/,例如www.a.com是可以闭合的,www.a.com/是不能够闭合的]
>后缀限制：使用?当作参数绕过 使用#当作锚点绕过 使用%20绕过

0x04 文件上传
====
在php中一般用3种方式实现文件的写入：

>move_uploaded_file
>fwrite
>file_put_contents

其中file_put_contents是最tm省事的，**file_put_contents(filename,contents)**就行了。
只要两个参数都可控你就有个php的webshell了。

其次是fwrite(fp,content),$fp是fopen函数返回的文件指针，所以只要fopen中的参数和content可控，你依然可以顺利的getshell

略麻烦的就是move_uploaded_file，需要读取tempfile然后复制到目标位置，和黑盒的文件上传一样，就不再赘述了。

0x05 命令执行
====
在php中执行php代码的函数有很多很多很多。。

我们常常写webshell时会使用eval，assert这样的

php中如果想动态调用一个函数可以使用call_user_func或者call_user_func_array，如果所有参数可控也是可以getshell的。

还有match_replce的\e模式也是可以造成rce的，~~总结不过来了就这样吧~~

0x06 杂七杂八的Tricks
====
往往代码审计中难以控制变量是白盒的一大难题，有些cms会使用伪全局的方法来注册变量或者在某些地方使用extract $$等方式动态声明变量。  
extract其实有两个参数，第二个参数用于声明是否覆盖已有变量之类的，很多开发者都不知道。  
全局变量就不说了，太古老了没什么可说的。  
php的弱类型一直是ctf出题人最喜欢玩的点，从==到false === false，赛棍们总是玩不腻，P牛也总结过相关的知识，可以挖掘一下他的博客。  
就先这样吧，有想继续写的再继续写。





