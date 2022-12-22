---
layout: post
title: 从Python的exec()聊起
date: 2017-06-10 19:53:02
categories: 原创
---


`exec()`是Python的built-in函数。其作用很好描述，就是执行以string类型存储的Python代码。话不多说举个例子。

```python
>>> i = 2
>>> j = 3
>>> exec("ans = i + j")
>>> print("Answer is: ", ans)
Answer is:  5
>>>
```

在上个例子里面，ans变量并没有显式的定义，但仍然可以在print函数中调用。这是exec语句执行了`"ans = i + j"`中的代码，定义了ans变量。

乍一看，这个功能很像C语言里的define宏定义：都是在代码里面插入可变的代码段。但其实还不一样，再看一个例子。

```python
>>> i = 5
>>> j = 7
>>> n = 0
>>> while n < i:
...     print("looping")
...     if j > i:
...         break
...     n += 1
... 
looping
>>> 
```

假设使用exec函数。


```python
>>> i = 5
>>> j = 7
>>> n = 0
>>> while n < i:
...     print("looping")
...     exec("""if j > 5:
...           \n    break""")
...     n += 1
... 
looping
Traceback (most recent call last):
  File "<stdin>", line 4, in <module>
  File "<string>", line 3
SyntaxError: "break" outside loop
```

在这里，exec函数为什么失效了呢?

根据Python文档，解释器会在执行到`break`语句时，会跳出离该句最近的while、for循环，如果解释器无法找到while、for循环，就会报错。因此，此处报错，说明了Python解释器没有找到exec之前的while循环。

实际上，仔细看文档会发现，解释器遇到exec函数时，会独立执行字符串内的语句。如果还有传参，那都是定义变量的字典。解释器，不会寻找字符串外的语法结构。也就是说，在这个例子中，解释器会独立执行语句

```python
if j > i:
    break
```

难怪，解释器会报错了。

而C语言完全不存在这个问题。

```C
#include <stdio.h>
#define JUDGE(x, y) if ((x) > (y))\
    break;
int main() {
    int i = 5;
    int j = 7;
    int n = 0;
    while (n < i) {
        printf("looping\n");
        JUDGE(j, i)
        n++;
    };
    
    return 0;
}
```

编译以后运行一下，看看。

```shell
$ gcc test.c -o test 
$ ./test 
looping
$
```

两个表面看上去类似的功能背后的原理完全不同。C语言的define，会在编译的第一步——“预处理”中完成替换。编译器在后续语法分析时，完全不知道原始代码里的宏定义是什么样子。

# 多说两句

## exec可以帮助完成过程抽象

exec是一个比较偏门的函数，而且过多地使用这个函数会降低代码的可读性。

不过，它有助于在开发过程中循序渐进的完成“过程抽象”。

最近，工作中就遇到一个场景：解析不同语言的代码文件。代码文件大致一样，却又随着语言语法的不同而在解析细节上有着不一样。我在最初拿到这个任务时，对于如何抽象出类和对象完全没有头绪。就先对不同的代码文件，单独写一个过程函数。全部写完，单元测试跑过之后，再去对比：归纳出共有的方法，提取共同的结构作为父类的内容。用exec函数剥离代码，分离出子类的私有方法。

## C语言中实现面向对象

** 以下内容误导性极大，笑笑就好，当年好菜啊。  **

C语言是典型的面向过程语言，怎么做到面向对象呢？

这个脑洞有点大，但是大牛们已经在这么做了。之前看知乎上有人讲：真正的C语言用家，都是用宏定义来实现类似于C++的面向对象特性。当初看了也是一头雾水，这次才悟出来怎么实现。

随便举个例子吧。

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MALE 0
#define FEMALE 1

#define INIT_PERSON(Person_var, Gender, Birthday)\
    struct Person* Person_var;\
    Person_var = malloc(sizeof(Person_var));\
    Person_var->gender = (Gender);\
    strcpy(Person_var->birthday, Birthday);

struct Person {
    int gender;
    char birthday[];
};

int main() {
    INIT_PERSON(li_ming, MALE, "1992-02-13")
    printf("Li ming gender is %d\n", li_ming->gender);
    printf("Li ming birthday is %s\n", li_ming->birthday);

    INIT_PERSON(han_mei_mei, FEMALE, "1989-09-21")
    printf("Han Meimei gender is %d\n", han_mei_mei->gender);
    printf("Han Meimei birthday is %s\n", han_mei_mei->birthday);
    
    free(li_ming);
    free(han_mei_mei);
    return 0;
}
```

代码中，`INIT_PERSON`宏就实现了：类似于面向对象中创建实例的方法。Person结构体，对应的类方法可以使用类似的方式来实现。

父类和子类的继承呢？当然可以通过递归调用宏定义来实现了。

当然了，以上只是一些粗浅的理解，这个方向还有很多细节可以挖掘。








