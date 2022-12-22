---
layout: post
title: "HtDP学习笔记 - 可变数据结构的编程"
date: 2016-03-12 19:53:02 +8000
categories: 笔记
---

# 可变数据结构

本书的第一部分主要介绍了程序预定义数据结构以及自定义数据结构的编程。这两类数据结构的共同点是，其内部数据域的数目是有限的，因此称作**固定数据(Fixed-Sized Data)**。但是，生活中往往还存在很多数据域数无法预先确定的情形。比如要统计某大学全校学生的身体健康情况，工作人员根本无法准确知道此次调查能够覆盖多少人次。本书将这类数据称作**可变数据(Arbitrarily Large Data)**。

本书引入了**链表**来表达这一类型的数据。这一概念类似于Python的`List`类型。链表是常见的数据结构之一，此处不再做介绍。

链表的特点之一是*自己指向自己*，也就是链表每个元素的`next`域都包含了指向自身类型的指针。因此也被称作**递归数据结构(Self-Referential Data)**。

# 递归函数

本书目前采用的BSL+语言中，没有`for`循环语句。唯一的逻辑选择语句只有`cond`表达式，类似于Python中的`if`语句。因此，要想遍历数组中的所有元素，必须另辟蹊径。本书为解决遍历数组问题引入了**递归函数(Self-Referential Function，或者Recursive Function)**。形式如下：

```python
def do_something_with(LIST):
	if <the LIST is empty>:
		<do something A>
	else:  # the list is not empty
		<do something B with the first element of LIST>
		do_something_with(<the rest of the LIST>)
```

递归函数有两个要点：

1. 递归就是在过程或函数里面调用自身；
2. 在使用递归时，必须有一个明确的递归结束条件，称为*递归出口*。

递归分为两个阶段：

1. 递推——把复杂的问题的求解推到比原问题简单一些的问题的求解；
2. 回归——当获得最简单的情况后，逐步返回，依次得到复杂的解。

P.S. 递归函数的有效性可以用数学归纳法来进行严格证明。具体过程可见[《计算机科学的基础》第2章: 迭代、归纳和递归](http://www.ituring.com.cn/tupubarticle/5504)。

# 何时设计辅助函数

要处理引用自身的数据(self-reference)需要设计递归函数，要处理交叉引用的数据(cross-reference)需要设计辅助函数。

具体的，遇到下列情况，建议使用辅助函数：

1. 当处理数据需要用到其他特定领域的知识，如图片合成、科学计算等等；
2. 当需要分情况处理数据时，应该设计一套`if`选择流程。如果这套流程很复杂，请单独设计为辅助函数；
3. 如果带处理数据中一部分是递归数据，应该将这部分数据的处理流程分离出来，形成辅助函数；
4. 如果以上方法无效，也许该设计一个more general函数（MGF，更泛的？？），并将现在的主函数设计为MGF的一个子类函数。
    
	>If everything fails, you may need to design a more general function and define the main function as a specific use of the general function. This suggestion sounds counter-intuitive but it is called for in a remarkably large number of cases.
 








