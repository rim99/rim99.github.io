---
layout: post
title: "HtDP学习笔记 - 如何自定义数据类型"
date: 2016-03-09 19:53:02 +8000
categories: 笔记
---


>本文主要摘译自[HtDP第五章](http://www.ccs.neu.edu/home/matthias/HtDP2e/part_one.html)，并结合Python语言的实际情况略作修改。原有语言DrRacket改写为Python。

通常的，编程语言都会提供基本的预定义数据类型，比如整型、浮点型、字符串类型等等。但是，当用编程来解决实际问题时，我们往往会发现这些很难满足需求。比如，我们需要建立一个公司的员工数据模型，其中包含姓名，年龄，性别，职务，联系手机等信息。这个模型包含了五条信息，根本没法用语言预先定义的数据类型来表达。

幸运的是，编程语言还提供了自定义数据类型的方法：面向过程的语言，如C，可以自定义一个结构体(Struct)；面向对象的语言，如Python可以自定义一个类(Class)。

```python
>>> class Personel_Info(object):
...     def __init__(self, name, age, gender, post, cellphone_no):
...             self.name = name
...             self.age = age
...             self.gender = gender
...             self.post = post
...             self.cellphone_no = cellphone_no
... 
```

实际上，定义一个数据类型的同时，也还定义了三类函数：

* 构造函数(constructor)，使用这一函数能够继续构造同样类型的数据实例(instance)。例如，李明，29岁，男，经理，手机号码：29900123456。
	
	```python
	>>> li_ming = Personel_Info("Li Ming", 29, "Male", "Manager", 29900123456)
	>>> li_ming
	<__main__.Personel_Info object at 0x10e13b990>
	```
	
* 指针函数(selector)，使用这一函数能够对数据实例的具体某一项执行取赋值操作。下例展示了对`li_ming`的`name`项的取赋值操作。
	
	```python
	>>> li_ming.name
	"Li Ming"
	>>> li_ming.name="Wang Lili"
	>>> li_ming.name
	"Wang Lili"
	```
	
* 判断函数(structure predicate)，使用这一函数能够判断某一数据是否为所定义的类型。Python使用`instance()`函数判断，数据实例和数据类型为该函数的参数。
	
	```python
	>>> isinstance(li_ming, Personel_Info)
	True
	```

# 编写数据类型的说明

之前的例子有个小瑕疵：**缺乏说明**。

试想一下，如果数据录入人员不清楚`age`的类型，有可能会录入字符串`"29"`，如果再换一个人，又可能录入了数字类型`29`。这些差异会对后期数据处理造成麻烦：得先判断是什么类型，然后执行下一步指令。从而，严重拖累程序效率。

这是本书推荐的数据类型说明样例：

```python
class Personel_Info(object):
	"""
	A Personel_Info is a class: Personel_Info( str, int, str, str, int)  # 写出构造函数，参数中注明数据类型，可以是语言预定义的类型，也可以是之前定义好的自定义类型
	
	Interpretation : Represent a person of the company.  # 注明该数据类型所表达的实际意义
	"""
	
    def __init__(self, name, age, gender, post, cellphone_no):
            self.name = name
            self.age = age
            self.gender = gender
            self.post = post
            self.cellphone_no = cellphone_no 
```

如有必要，还应该在说明中举例：

1. 如果是语言的预定义类型，数字、字符串、布尔值等，可以直接给出自己喜欢的例子；（有些指南会建议取一个描述性的名字，然而还是比不上好的数据类型说明）
2. 对于枚举类型，可以从中选择若干；
3. 对于区间类型，应该给出边际值，如果包含的话，同时再给出一个区间中间值；
4. 如果是选择结构类型（类似于数学中的分类函数），应该各自举例；
5. 对于自定义类型，就给出一个数据实例。





























