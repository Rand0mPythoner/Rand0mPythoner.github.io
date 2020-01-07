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

0x03 find_element_by_*
==
selenium浏览器类提供了搜索元素的函数，总的来说还是比bs4简洁许多，门槛低了很多，直接调用方法就可以返回一个元素对象
```
 def find_element_by_id(self, id_):
        
        return self.find_element(by=By.ID, value=id_)

    def find_elements_by_id(self, id_):
        
        return self.find_elements(by=By.ID, value=id_)

    def find_element_by_xpath(self, xpath):
        
        return self.find_element(by=By.XPATH, value=xpath)

    def find_elements_by_xpath(self, xpath):
        
        return self.find_elements(by=By.XPATH, value=xpath)

    def find_element_by_link_text(self, link_text):
        
        return self.find_element(by=By.LINK_TEXT, value=link_text)

    def find_elements_by_link_text(self, text):
        
        return self.find_elements(by=By.LINK_TEXT, value=text)

    def find_element_by_partial_link_text(self, link_text):
        
        return self.find_element(by=By.PARTIAL_LINK_TEXT, value=link_text)

    def find_elements_by_partial_link_text(self, link_text):
        
        return self.find_elements(by=By.PARTIAL_LINK_TEXT, value=link_text)

    def find_element_by_name(self, name):
        
        return self.find_element(by=By.NAME, value=name)

    def find_elements_by_name(self, name):
        
        return self.find_elements(by=By.NAME, value=name)

    def find_element_by_tag_name(self, name):
        
        return self.find_element(by=By.TAG_NAME, value=name)

    def find_elements_by_tag_name(self, name):
        
        return self.find_elements(by=By.TAG_NAME, value=name)

    def find_element_by_class_name(self, name):
        
        return self.find_element(by=By.CLASS_NAME, value=name)

    def find_elements_by_class_name(self, name):
        
        return self.find_elements(by=By.CLASS_NAME, value=name)

    def find_element_by_css_selector(self, css_selector):
        
        return self.find_element(by=By.CSS_SELECTOR, value=css_selector)

    def find_elements_by_css_selector(self, css_selector):
        
        return self.find_elements(by=By.CSS_SELECTOR, value=css_selector)
```
可以看出来全都调用了find_element和find_elements方法，跟进去学习一下：
```
 def find_element(self, by=By.ID, value=None):
        if self.w3c:
            if by == By.ID:
                by = By.CSS_SELECTOR
                value = '[id="%s"]' % value
            elif by == By.TAG_NAME:
                by = By.CSS_SELECTOR
            elif by == By.CLASS_NAME:
                by = By.CSS_SELECTOR
                value = ".%s" % value
            elif by == By.NAME:
                by = By.CSS_SELECTOR
                value = '[name="%s"]' % value
        return self.execute(Command.FIND_ELEMENT, {
            'using': by,
            'value': value})['value']
```
我把注释去了，因为没啥可解释的，就是一堆if，execute这个函数是一个动态调用，call到command.FIND_ELEMENT，然后传入了一个字典，最后取value  
进入FIND_ENELMENT瞅瞅。  
``` Python
FIND_ELEMENT = "findElement"
	Command.FIND_ELEMENT: ('POST', '/session/$sessionId/element')
```
发现是个API，看来是我猜错了，还是要看一下execute函数
```
def execute(self, command, params):
    command_info = self._commands[command]
    assert command_info is not None, 'Unrecognised command %s' % command
    path = string.Template(command_info[1]).substitute(params)
    if hasattr(self, 'w3c') and self.w3c and isinstance(params, dict) and 'sessionId' in params:
        del params['sessionId']
    data = utils.dump_json(params)
    url = '%s%s' % (self._url, path)
    return self._request(command_info[0], url, body=data)
```
最后发现还是调用了_request方法，data就是  
```
{'using': by,'value': value}
```
原理就分析到这里了，下一章会用一个爬虫来演示selenium怎么使用。