---
layout: post
title: "Python selenium 随手记1"
subtitle: "Python selenium note .1"
author: "P1an0"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - python
  - selenium
---

> 因为工作需要所以开了selenium这个坑，这个东西打算从会用到搞懂原理（因为很强大）

  selenium是python一款可以模拟浏览器操作的库，可以模拟各种浏览器做人工操作，是一款非常强大且热门的工具。

  selenium的坑我好长时间就想开了，因为看同事用发现太好用了，对于半自动化渗透测试也有帮助，这里只带一遍googleChrome的操作，不一定百分百和其他浏览器雷同，但是大同小异，咱们开始8！

0x01 selenium的安装:
========
​	selenium安装非常简单，只需要pip install selenium就可以了。

​	在安装selenium之后，还需要安装对应chrome版本的selenium，查看chrome版本可以在浏览器地址栏输入:  
​	chrome://version  
>Google Chrome	**79.0.3945.88** (正式版本) （64 位） (cohort: Stable)

加粗的就是chrome的版本啦，最后两位是小版本可以忽略，我们去chrome的官网找到对应版本的chromeDriver
https://chromedriver.storage.googleapis.com/index.html?path=  
找到对应版本下载即可。
selenium也有中文的官方文档：  
`https://python-selenium-zh.readthedocs.io/zh_CN/latest`

0x02 selenium初探:
========
引入selenium很简单，只需要  
```from selenium import webdriver```
一个简单的selenium程序：
```python
from selenium import webdriver
dirver = webdriver.Chrome()
dirver.get("http://Rand0mPythoner.github.io/")
```
模拟浏览器对web控件的操作需要使用  
```from selenium.webdriver.common.keys import Keys```
使用 driver.find_element_by_name({**标签的Name属性**})来获取一个标签对象
在使用send_keys()来进行操作
官方文档中的*assert*(断言)较为复杂，这里不用了。