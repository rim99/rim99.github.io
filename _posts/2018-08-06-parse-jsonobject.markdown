---
layout: post
title: 考虑实现一个JSON Parser
date: 2018-08-06 14:54:29
categories: 原创
---

最近时间比较多，认真看了一下龙书。当然还没看完，但是看到一半就发现：编译前端的知识正好能够完成解析JSON字串了么。

首先，看一下JSON的结构。

JSON结构包括JsonObject和JsonArray两种格式，两者可以相互嵌套。前者是一种无序key-value映射，像Java里的Map；后者是一种有序线性结构，像Java里的List。

JsonObject的key只能是字符串，value可以是字符串String，数字Integer或Float，布尔值Boolean，或者干脆是个空null，当然也可以是JsonObject或JsonArray。String值必须以`"`包裹，key和value以`:`分隔，key-value映射之间必须以`,`分隔。

JsonArray里的每一个元素都是一个值。值之间必须以逗号`,`分隔。


上面这一串话可以写成一组[上下文无关文法](https://cn.bing.com/search?q=%E4%B8%8A%E4%B8%8B%E6%96%87%E6%97%A0%E5%85%B3%E6%96%87%E6%B3%95)，如下所示。


```
# S为开始符号，O为JsonObject，A为JsonArray
# S可以是JsonObject也可以是JsonArray
S -> O 
   | A

# JsonArray由多个值V组成，也可以为空E
A -> [ V,V, ... ] 
   | E

# JsonObject由多个键值对KV组成，也可以为空E
O -> { KV, KV, ... } 
   | E

# 键值对包含一个key和一个值V，key必须是str类型
# 为了容错，键值对也可以为空E
KV -> str:V
    | E

# 值可以是多种类型
V -> str 
   | int 
   | float 
   | bool 
   | null 
   | A 
   | O

# 给出各个基础类型的正则表达式
str -> "(.)+"
int -> \d+
float -> \d+.\d+
bool -> true|false
null -> null
E -> \s+

```

正则表达式的规则来自于[API doc - java.util.regex.Pattern](https://docs.oracle.com/javase/10/docs/api/java/util/regex/Pattern.html)。


第二步，就是翻译制导方案，其实也就可以生成JSON对象了。

程序不断地读取输入流。

每读到一个`{`，就生成一个`Map<String, Object>`对象，作为一个JsonObject。

然后识别其中的键值对KV：确认以`"`开头，读取到下一个`"`时，则将这之间读取到的字符作为一个key。此处，必须有一个flag，便于确认当前缓冲区的字符是否为字符串的一部分。每一次读到一个`"`，flag就翻转一次(true变false，false变true)。当然得确保这个`"`之前没有转义符号`\`。

读到`:`，说明该读取值了。读完值，将键值对插入字典。

读到`}`，说明JsonObject读取完毕。

类似的，每读到一个`[`，就生成一个`List<Object>`对象，作为一个JsonArray。

读到`]`，说明JsonArray读取完毕。

这里说说思路，懒癌又犯了，代码就不写了。
