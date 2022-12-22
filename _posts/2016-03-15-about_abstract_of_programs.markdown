---
layout: post
title: "Composing Programs学习笔记 - 抽象"
date: 2016-03-15 19:53:02 +8000
categories: 笔记
---


设计程序时，会经常发现很多代码都是大同小异的。为了让程序更简洁，也便于代码的修改，实现**代码复用**是非常重要的。

将代码段的重复部分提取出来，差异部分作为新函数的参数。这个过程就称作**抽象**，根据程序代码的实际用途，也可以分为**函数抽象**和**数据抽象**。

# 如何设计抽象

1. 寻找代码的相似之处。
2. 对相似度足够高的代码，把相同的部分提取出来作为新函数。然后标出其中的不同之处，再将这些不同列作*参数*，并根据这些参数调整新函数。
3. 编写函数的说明。
4. 测试新函数。

这是函数抽象的步骤，数据抽象也是类似。

# 如何使用抽象

1. 根据实际问题，写出函数的函数签名，目标陈述，函数存根，测试样本。

	>... distill the problem statement into a signature, a purpose statement, an example, and a stub definition.
 
2. 找出匹配的抽象。匹配的意思是，该抽象的目标陈述囊括了目标函数(more general)，而且签名相类似，
3. 写出函数模版。
4. 设计新函数内部的局部辅助函数。DrRacket里需要用到`local`表达式.
5. 测试。

# 关于Python中的抽象

Python和DrRacket有很大的不同。

以上只是理论，具体操作可以看[Composing Programs](http://composingprograms.com)。









