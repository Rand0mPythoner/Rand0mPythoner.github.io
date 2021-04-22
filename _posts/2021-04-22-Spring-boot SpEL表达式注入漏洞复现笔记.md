---
layout: post
title: "Spring-boot SpEL表达式注入漏洞复现笔记"
subtitle: "WTF about Spring-boot SpEL"
author: "Rand0m ^ Novocaine#p1an0@qq.com"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - Java
  - spring
---

0x01 前言
--
- 最近hvv太累了（根本没参加啊喂
- 休养生息了一段时间，一直在想后面该学什么完成什么。
- 找几个比较有用的简单的漏洞复现分析一下吧。
- 这波没有行号了，Spring每个小版本行号都乱七八糟的，直接贴代码兄弟萌过把瘾。

0x02 漏洞周边
--
 - 影响版本：
   - 1.1.0-1.1.12
   - 1.2.0-1.2.7
   - 1.3.0
 - 修复方案：升至1.3.1或以上版本
 - 本文环境：1.3.0
 - 漏洞披露时间：2016
 - 黑盒利用条件：有接口触发Spring 默认错误页
 - CVE：应该是没有
 - POC：${T(java.lang.Runtime).getRuntime().exec(new String(new byte[]{0x63,0x61,0x6c,0x63}))}

0x03 环境搭建
--

Spring-boot 1.3.0

https://github.com/spring-projects/spring-boot/releases/tag/v1.3.0.RELEASE

0x04 漏洞分析:
--
根据其他dalao的文章看到漏洞触发在
```
spring-boot-autoconfigure\1.3.0.RELEASE\spring-boot-autoconfigure-1.3.0.RELEASE.jar!\org\springframework\boot\autoconfigure\web\ErrorMvcAutoConfiguration.class
```
SpelView类初始化：
```
    private static class SpelView implements View {
        private final String template;
        private final StandardEvaluationContext context = new StandardEvaluationContext();
        private PropertyPlaceholderHelper helper;
        private PlaceholderResolver resolver;

        SpelView(String template) {
            this.template = template;
            this.context.addPropertyAccessor(new MapAccessor());
            this.helper = new PropertyPlaceholderHelper("${", "}");
            this.resolver = new ErrorMvcAutoConfiguration.SpelPlaceholderResolver(this.context);
        }
```
this.template = template;
定义模板
模板是这样：
```html
<html><body><h1>Whitelabel Error Page</h1><p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p><div id='created'>${timestamp}</div><div>There was an unexpected error (type=${error}, status=${status}).</div><div>${message}</div></body></html>
```

this.helper = new PropertyPlaceholderHelper("${", "}");
定义了表达式的结构

this.resolve.contextr 取值就是this.context

跟过去在render函数下断：
```java
public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
            if (response.getContentType() == null) {
                response.setContentType(this.getContentType());
            }

            Map<String, Object> map = new HashMap(model);
            map.put("path", request.getContextPath());
            this.context.setRootObject(map);
            String result = this.helper.replacePlaceholders(this.template, this.resolver);
            response.getWriter().append(result);
        }
```
this.context.rootObject是一个map对象，map对象中的message内容是我们写入的payload
this.helper.replacePlaceholders方法传入模板和this.reslover
刚才说this.reslover.context = this.context,所以this.reslover.context.rootObject也是同样的map对象。

跟进this.helper.replacePlaceholders
进入org\springframework\util\PropertyPlaceholderHelper.class

```java
 public String replacePlaceholders(String value, PropertyPlaceholderHelper.PlaceholderResolver placeholderResolver) {
        Assert.notNull(value, "'value' must not be null");
        return this.parseStringValue(value, placeholderResolver, new HashSet());
    }
```
跟！
```java
protected String parseStringValue(String strVal, PropertyPlaceholderHelper.PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {
        StringBuilder result = new StringBuilder(strVal);
        int startIndex = strVal.indexOf(this.placeholderPrefix);

        while(startIndex != -1) {
            int endIndex = this.findPlaceholderEndIndex(result, startIndex);
            if (endIndex != -1) {
                String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
                String originalPlaceholder = placeholder;
                if (!visitedPlaceholders.add(placeholder)) {
                    throw new IllegalArgumentException("Circular placeholder reference '" + placeholder + "' in property definitions");
                }

                placeholder = this.parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
                String propVal = placeholderResolver.resolvePlaceholder(placeholder);
...
```

下面两个if我没截，不看也没啥影响

strVal是我们最开始传入的this.template，就是我们表达式的模板，第二个变量是placeholderResolver，我们传入的this.reslover。

placeholder = this.parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);

这地方是个递归，会递归解析placeholder中的表达式。

下面一大段if乱七八糟的就是把${123123}解析为123123然后传给placeholder变量（脱马甲？

注意下一句：
```java
String propVal = placeholderResolver.resolvePlaceholder(placeholder)
```

跟！

```java
public String resolvePlaceholder(String name) {
            Expression expression = this.parser.parseExpression(name);

            try {
                Object value = expression.getValue(this.context);
                return HtmlUtils.htmlEscape(value == null ? null : value.toString());
            } catch (Exception var4) {
                return null;
            }
        }
```
看到Object value = expression.getValue(this.context);

解析传入的表达式，取出表达式执行结果还给value变量。

这里就应该是rce的最后一环了，调试，单步，看到计算器，结束了。

这波没有代码diff环节了，这个文件已经不存在于spring-boot中了。。。

0x05 
--
补丁分析：
SpelView类初始化时this.helper参数用了一个新写的类，那个类没分析，核心概念就是避免递归。

0x06 参考文档
--
-  https://www.cnblogs.com/litlife/p/10183137.html
-  https://github.com/spring-projects/spring-boot
