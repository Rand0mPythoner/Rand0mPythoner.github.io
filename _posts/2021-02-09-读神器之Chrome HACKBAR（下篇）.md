---
layout: post
title: "读工具之HACKBAR（下篇）"
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

0x02 接上篇
--

还剩下其他模块，代码太串了，感觉二开也是个力气活QAQ

后面除了mysql的语句生成以外，都使用了applyFunction动态调用，这里只分析一次applyFunction，我在关键行后面加了中文注释方便理解

```javascript
      applyFunction: function (func, insertWhenNoSelection = false, argument = undefined) {
        func = this.getNamespaceByPath(func, window, true) //获取函数命名空间

        if (this.domFocusedInput === null) {
          return
        }

	//取输入框的东西
        let startIndex = this.domFocusedInput.selectionStart 
        let endIndex = this.domFocusedInput.selectionEnd
        const textSelected = (endIndex - startIndex !== 0)
        const inputText = this.domFocusedInput.value
...
 try {
          processed = func.namespace[func.name](argument) //函数调用
        } catch (error) {
          this.snackbar.text = error.message
          this.snackbar.show = true
        }

document.execCommand('insertText', false, processed)  //把结果扔到输入框

```

### Sqli:

pyload都在/scripts/lib/payload.js中
重复的东西太多，先不看了

### xss
```javascript
//核心就这些 payload.js
window.Payload.XSS = {
  polyglot: value => "jaVasCript:/*-/*`/*\\`/*'/*\"/**/(/* */oNcliCk=alert() )//%0D%0A%0D%0A//</stYle/</titLe/</teXtarEa/</scRipt/--!>\\x3csVg/<sVg/oNloAd=alert()//>\\x3e"
}
```

### LFI
```javascript
//核心就这些 payload.js
window.Payload.LFI = {
  phpWrapperBas64: value => 'php://filter/convert.base64-encode/resource=' + value
}
```

### SSTI
```javascript
payload.js
window.Payload.SSTI = {
  flaskRCE: value => {
    // Reference: https://twitter.com/realgam3/status/1184747565415358469
    return "{{ config.__class__.__init__.__globals__['os'].popen('ls').read() }}"
  }
```

以上是生产Payload的地方，值得一提的是我们想要二开可以直接在下面添加payload，模仿index.html的写法就可以添加模块。

### EN/DECODING

方法写在了lib/encode.js和hash.js
大多都是用CryptoJS.enc插件实现的,在二开我会尝试把base64变成自定义密钥的形式

0x03 总结
--
Hackbar在渗透过程中的确是一件趁手的兵器,感觉chrome这个插件的作者同样也是二开的,因为基础功能代码风格明显和后面的功能不太一样,下面是我想给它打造的新功能:
 - 字符fuzz 
 - base64换密钥编解码(保留原功能)
 - 更多的payload生成
   - 一句话木马
   - 各种反弹shell指令
   - 各种 Ndays的payload等
