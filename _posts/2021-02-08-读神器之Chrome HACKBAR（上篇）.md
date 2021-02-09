---
layout: post
title: "读工具之HACKBAR（上篇）"
subtitle: "WTF about HACKBAR"
author: "Rand0m ^ Novocaine#p1an0@qq.com"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - javascript
  - ChromeExt
---

0x01 序
--
- 一直很敬仰某师傅，反复看了某师傅的带我读神器系列，发现阅读这些代码可以学习作者赋予工具的灵魂及对安全的理解，所以以后会有读神器甚至二开系列。
- 学习一些复杂的功能实现
- 先从简单的工具开始读，然后慢慢地解剖巨人的肩膀。

0x02 HackBar
--

### 环境安装

代码下载：
 - git clone https://github.com/0140454/hackbar.git

注意自己找的时候别找错了，HackBar有两个版本，我使用的是图标是一个H的HackBar版本

### manifest.json
Chrome插件先看一个manifest.json
我这边摘了有用的字段
```json
...
  "devtools_page": "devtools.html",
...
 "permissions": [
    "tabs",
    "storage",
    "webRequest",
    "webRequestBlocking",
    "<all_urls>"
  ],
...
	"background": {
    "scripts": ["scripts/background.js"]
  },
  "web_accessible_resources": [
    "payloads/*"
  ],
  ...
```

分解：
 - devtools_page
   
   -  开发者模式的界面
 - permissions[]
   
   - 申请的权限（TAB页，存储，发送请求，阻塞请求，所有url都生效） 
 - background:
   
   - 引用的后台脚本/页面（scripts/background.js）
 - web_accessible_resources
   - 配置目录访问权限，payloads目录下的文件都可以被插件访问

### 目录结构
 - payloads/			目录扫描功能要用的字典，可以改
 - scripts/				 引用的Js代码
   - /crypto-js/		    加解密的轮子
   - /lib/				一些功能放在里面
   - /test/				目录测试模块
   - /vue.js/			  vue
   - /background.js		后台逻辑代码
   - /devtools.js		 开发者模式js
   - /index.js			定义了vue和全局方法
 - devtools.html		  开发者模式的界面（只引用了devtools.js）
 - index.html			真正的开发者模式界面，create in /script/devtools.js
 - manifest.json		  不解释了

### 阅读

这里避免篇幅太长，直接读各个功能的逻辑，我把核心代码逻辑直接摘出来供阅读。

先看看功能有哪些（第一个我直接管他叫发包模块各位没异议吧）：

![image-20210208165855434](https://raw.githubusercontent.com/Rand0mPythoner/imgs/master//blog-imgimage-20210208165855434.png)

#### 发包模块
三个按钮 load,split,execute

 - load:
```javascipt
index.js:69
load: function () {
        this.backgroundPageConnection.postMessage({
          tabId: chrome.devtools.inspectedWindow.tabId,
          type: 'load'
        })
      },
...
background.js:11
  if (message.type === 'load') {
    tabDB[message.tabId].connection.postMessage({
      type: 'load',
      data: tabDB[message.tabId].request
    })
  }
...
index.js:227
if (message.type === 'load') {
          if (typeof message.data === 'undefined') {
            this.reloadDialog = true
            return
          }

          const request = message.data

          this.request.url = request.url
          this.request.body.enabled = (typeof request.body !== 'undefined')
          if (typeof request.contentType !== 'undefined') {
            this.request.body.enctype = request.contentType.split(';', 1)[0].trim()
          }

          if (this.request.body.enabled) {
            if (typeof request.body.formData !== 'undefined') {
              const params = new URLSearchParams()

              for (const name in request.body.formData) {
                request.body.formData[name].forEach(value => {
                  params.append(name, value)
                })
              }

              this.request.body.content = params.toString()
            } else {
              this.request.body.content = ''

              request.body.raw.forEach(data => {
                if (typeof data.file !== 'undefined') {
                  this.request.body.content += `[Content of '${data.file}']`
                } else {
                  this.request.body.content += data.bytes
                }
              })
            }
          }

          this.$refs.url.focus()
        }
```
index.js：227之前都是把请求发送到后台
之后的流程就是：

 - line 228的判断是判断传入的url是否为空
 - line 237处理content-type
 - line 242处理POST的DATA
 - line 225处理raw DATA

 - split:
```javascript
index.js:76
split: function () {
        this.request.url = this.request.url.replace(/[^\n][?&#]/g, str => {
          return str[0] + '\n' + str[1]
        })

        if (typeof this.request.body.content !== 'undefined' &&
          this.request.body.enctype !== 'multipart/form-data') {
          this.request.body.content = this.request.body.content.replace(
            /[^\n][?&#]/g, str => {
              return str[0] + '\n' + str[1]
            })
        }
      },
```
找到?&#符号加个换行符，没什么好说的

 - execute
 ```javascript
 index.js:90
       execute: function () {
        if (this.request.url.length === 0) {
          return
        }

        this.backgroundPageConnection.postMessage({
          tabId: chrome.devtools.inspectedWindow.tabId,
          type: 'execute',
          data: this.request
        })
      },
 ...
 background.js:16
if (message.type === 'execute') {
    tabDB[message.tabId].modifiedHeaders = message.data.headers

    if (message.data.body.enabled) {
      if (enctypeNeededToOverrideHeader.indexOf(message.data.body.enctype) >= 0) {
        tabDB[message.tabId].modifiedHeaders.unshift({
          enabled: true,
          name: 'content-type',
          value: message.data.body.enctype.split(' ', 1)[0]
        })
      }

      chrome.tabs.executeScript(message.tabId, {
        file: 'scripts/lib/post.js'
      }, () => {
        chrome.tabs.sendMessage(message.tabId, message.data, response => {
          if (response === null) {
            return
          }

          tabDB[message.tabId].connection.postMessage({
            type: 'error',
            data: response
          })
        })
      })
    } else {
      chrome.tabs.update(message.tabId, {
        url: message.data.url
      })
    }
  }

 ```



 - background.js:19 判断请求为GET/POST

 - background.js:28 处理POST数据，详情自己到post.js细看

 - background.js:31 发送POST请求

 - background.js:43 控制tab发送get请求

Test：
```javascript
                    <v-list-item @click="controlTest('start', 'scripts/test/paths.js', { payloadsPath: chrome.runtime.getURL('payloads/paths.txt'), againstWebRoot: true })">
                      <v-list-item-title>Against web root directory</v-list-item-title>
                    </v-list-item>
                    <v-list-item @click="controlTest('start', 'scripts/test/paths.js', { payloadsPath: chrome.runtime.getURL('payloads/paths.txt'), againstWebRoot: false })">
                      <v-list-item-title>Against current directory</v-list-item-title>
...
index.js:102
controlTest: function (action, script = undefined, argument = undefined) {
        if (action === 'start') {
          this.testProgressDialog.percentage = null
          this.testProgressDialog.status = null
          this.testProgressDialog.result = null
          this.testProgressDialog.error = null
          this.testProgressDialog.show = true
        }

        this.backgroundPageConnection.postMessage({
          tabId: chrome.devtools.inspectedWindow.tabId,
          type: 'test',
          data: { action, script, argument }
        })
      },
...
background.js:47
if (message.type === 'test') {
    if (message.data.action === 'start') {
      chrome.tabs.executeScript(message.tabId, {
        file: message.data.script
      }, () => {
        chrome.tabs.sendMessage(message.tabId, message.data)
      })
    } else {
      chrome.tabs.sendMessage(message.tabId, message.data)
    }
  }
}
//paths.js比较长，各位自己看吧
```
