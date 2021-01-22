---
layout: post
title: "HtDP学习笔记 - 如何设计程序"
date: 2016-03-05 19:53:02 +8000
categories: 笔记
---


>本文摘译自[HtDP第三章](http://www.ccs.neu.edu/home/matthias/HtDP2e/part_one.html)，原有语言DrRacket改写为Python。本文还混有[Composing Programs](http://composingprograms.com)的内容。

设计程序当中很重要的一个环节是**将问题转化为程序**。在这当中需要注意区分哪些对于描述问题至关重要，哪些无关紧要。此外，我们还应当确定目标程序的参数是什么，输出结果又是什么，以及参数与输出结果有什么样的内在联系。我们还应该注意所选择的编程语言以及相关库是否对数据处理过程提供了基本的运算支持。如果没有，那我们得自己编写相关的辅助函数。最后，一旦程序完成，我们得检验一下程序是否按照预想的方案执行计算过程。

程序写好后，最好有一个简短的说明，讲述这个程序的功能，所需要的参数，以及所产生的结果。如能够说明程序的正常运行状态就再好不过了。最好的情况是，程序与求解问题的对应关系非常精确。即如果求解问题状态有小幅变动，程序也同样能够对此作出小幅调整。

我们之所以作出上述要求是因为程序不止是满足客户要求能够运行即可。我们要考虑到程序后期维护的需要：如果团队人员发生变动，新员工必须尽快读懂程序。此外，提高程序的可读性，也会方便客户随时改动。

本书针对上述情况提出了一套系统地设计程序的流程。

# 设计函数

## 信息与数据

信息是对这个世界的描述。例如，三张电脑桌，价格分别为599元，1299元，2899元。
数据是对信息中与问题相关部分的抽象。例如，要求出三张桌子的平均价格，那么这三张桌子的品牌，材质就无关紧要，从而三张桌子可抽象为一个数组`tables = [599, 1299, 2899]`。

从某种意义上讲，程序实际上是对信息处理流程的描述。程序将真实世界的信息转化为数据，经过计算处理后，得出新的数据，然后将其转化为信息并输出。其信息的来源范围实际上是这个真实世界的一部分，称作**程序的定义域(Domain)**。

例如一个智能手机操作系统需要捕捉手指的点击，将触摸屏幕的电学信号转化为屏幕坐标。正在运行的app接收到坐标后，根据预先编好的程序，发出指令要求屏幕上的图片翻转一次。操作系统按照指令，控制屏幕的像素点的电学信号，不断的刷新屏幕，生成图片翻转动画。

软件工程界使用*MVC(Model-View-Controller)*模型来组织上述过程的代码。三个字母分别指代：数据处理过程、数据->信号过程、信号->数据过程。

![MVC](http://7xqan1.com1.z0.glb.clouddn.com/mvc.png)

## 撰写函数说明

当明白了信息与数据的相互转化过程后，就可以根据如下流程设计函数：

1. 使用注释语句解释一下如何使用数据描述信息。例如，
	
	```python
	"We use sequence to represent the price of tables"
	```
	
2. 写出函数签名(Function Signature)，目标陈述(Purpose Statement)，函数存根(Function Header, 也叫stub)
	
	**函数签名(Function Signature)**描述了函数所需参数和产生的结果。例如，一个将整型数组转化为整型数字的函数的签名为
	
	```python
	"Integer[] -> Integer"
	```
	**目标陈述(Purpose Statement)**描述了函数的作用：这个函数计算了什么。例如，一个计算三张桌子平均值的函数的目标陈述为
	
	```python
	"Calculate the average price of tables"
	```
	
	**函数存根(Function Header, 也叫stub)**将函数的参数替换为具体的合法数据，并提供了输出结果的具体示例
	
	```python
	"100 = average_price([int, int, int])" # average_price为函数名
	```
	
3. 使用样例来证明以上三项。
 
 	```python
	"""
   	>>> average_price([50, 100, 150])
   	100
   	>>> average_price([80, 160, 240, 320])
   	200
   	"""
   	```   
			    
4. 编写函数代码
	也就是完成对*信号->数据过程*、*数据处理过程*、*数据->信号过程*的描述。

5. 测试函数
	
	Python中可以使用`run_docstring_examples`函数来完成这个过程。
	
	```Python
    def average_price(prices):
		"""This function is to calculate the average price of tables.
    	    
		We use sequence to represent the price of tables.
    	    
		Integer[] -> Integer
    	    
		100 = average_price([int, int, int])
    	    
		>>> average_price([50, 100, 150])
		100
		>>> average_price([80, 160, 240, 320])
		200
		"""
		total = 0
		count = len(prices)
		for price in prices:
			total += price  
		average = total / count
		return average


	from doctest import run_docstring_examples
	run_docstring_examples(average_price, globals(), True)
   	```
    
    函数`average_price()`在声明时使用三个引号提供了一份简单的说明。在解释器中执行`help()`函数皆可获得这段说明（按Q退出）。同时，这份说明也指出了特定值下的输出值。利用`run_docstring_examples()`函数可以自动完成检验，并输出检验结果。如下,

	```python
	Trying:
		average_price([50, 100, 150])
	Expecting:
		100
	ok
	
	Trying:
		average_price([80, 160, 240, 320])
	Expecting:
		200
	ok
	```
	
	 

# 其他须知

计算机程序都是在解决实际问题，因此编程人员应当对程序所应用的相关学科有一定的了解，例如数学、音乐、生物、土木工程等等。

此外，还要对所使用的函数库的API有所了解。例如，要用Python处理IP相关问题，应当对[ipaddress函数库](https://docs.python.org/3/library/ipaddress.html)有所了解。

# 从函数到程序

复杂的程序不可能只有一个函数。大多程序都需要不少辅助函数，有些还需要定义很多常量。因此一定要使用好辅助函数和全局常量，来系统的设计好每一个函数。_记得给函数和常变量起一个有意义的名字_。

如果有必要，应该把预先定义的全局常量在函数说明里公示出来，以提醒程序维护人员这些常量的存在。

通常，随着程序编写的深入，你会发现需要不断添加新的辅助函数和常量。我们建议你在编写程序的过程同时也维护一张**“目标清单”**，把需要添加的新的辅助函数和常量列进清单。一旦完成，把它划去。只要清单上还有项目，就继续工作。如果你发现清单里的项目全部完成了，那编程的工作也就结束了。

# 程序测试

Python还可以使用`assert`声明执行成规模的自动化的函数测试。
	
```python
# define a function to test
def fib(x):
    if x==0:
		return  0
    elif x==1:
		return  1
    else:
		return fib(x-1)+fib(x-2)
	
# define a test function       
def fib_test():
	assert fib(2) == 1, "The 2nd Fibonacci number should be 1"
	assert fib(3) == 1, "The 3rd Fibonacci number should be 1"   
	assert fib(50) == 7778742049, "Error at the 50th Fibonacci number"
  
# execute test      
fib_test(2)
```











