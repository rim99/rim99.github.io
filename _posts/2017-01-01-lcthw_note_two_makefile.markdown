---
layout: post
title: 《笨办法学C》笔记之Makefile
date:  2017-01-01 19:53:02
categories: 笔记
---

# 使用gcc编译C语言源码

在Linux系统中，C语言源码需要用gcc编译为二进制可执行文件，才能够运行。

```bash
$ gcc test.c -o test
```

这句命令就将test.c文件编译为test二进制可执行文件。

```bash
$ ./test
```

如此可以直接执行编译后的test二进制可执行文件。

## 如何编译多个.c文件

**例1** 需要将test1.c、test2.c、test3.c合并编译为一个test可执行文件。

一种办法是：

```bash
$ gcc test1.c test2.c test3.c -o test
```

这个办法的缺陷是，每次会将所有.c文件编译一次。如果下次编译时，只有test3.c文件发生变动，那么重复编译test1.c和test2.c文件显得有些多余。

另一个办法则是：

```bash
$ gcc -c test1.c 
$ gcc -c test2.c 
$ gcc -c test3.c 
$ gcc -c test test1.o test2.o test3.o
```

gcc使用`-c`选项，可以将.c文件编译为.o对象文件。对象文件是gcc将源码编译为二进制文件的中间结果，省去了最后的*链接阶段*。

最后一行命令里，gcc将各个.o对象文件组合链接为完整的二进制可执行文件。

如果能够做到：

>在编译.o文件之前检查对应的.c文件的最后修改日期是否在.o文件的生成日期之后，如果是，才会再次编译

那整个编译过程会大大减少耗时。

而Make系统就可以做到这一点。

# Makefile

**例2** 考虑一个小型的C语言项目：

```bash
tmp/
   +---- include/
   |      +---- f1.h
   |      +----f2.h
   +----f1.c    #include "include/f1.h"
   +----f2.c    #include"include/f2.h"
   +---main.c   #include"include/f1.h", #include"include/f2.h"
```

对应makefile如下所示：

```makefile
#Makefile，Create testmf from f1.c f2.c main.c

all: main.o f1.o f2.o
        gcc -o testmf main.o f1.o f2.o
f1.o: f1.c
        gcc -c -o file1.o file1.c
f2.o: f2.c
        gcc -c -o file2.o file2.c
main.o
        gcc -c -o main.o main.c
clean:
        rm -rf f1.o f2.o main.o testmf

```

如果在tmp目录下直接执行`make all`，那么make系统首先会搜索`all`标签，并执行其对应的命令：`gcc -o testmf main.o f1.o f2.o`。接着，make会去递归查找这一命令对应的参数文件main.o、f1.o、f2.o：

* 如果文件不存在，直接执行对应的编译命令；
* 如果存在但对应.c文件已经更新，仍然会执行对应的编译命令；
* 如果存在而已经是最新，那么就会直接调用编译好的.o对象文件。

回过头来考虑例1——如果下次编译时，只有test3.c文件发生变动，如果调用makefile，那么整个过程仅仅会执行`gcc -c test3.c`和`gcc -c test test1.o test2.o test3.o`两个命令。

## 其他

1. 调用make命令后，系统会搜索Makefile或者makefile文件；也可以使用`make -f`指定自定义文件名。
2. 其实make后面的参数就是个makefile里的标签，至于标签里的填的是什么并不重要。把第一行的`all`改成`main`，那么执行`make main`就和原先的`make all`是同样的效果。
3. 可以使用`$()`来表示变量，方便构建[复杂的makefile](https://wizardforcel.gitbooks.io/lcthw/content/ex28.html)。

# 参考资料

* [gcc命令](http://man.linuxde.net/gcc)
* [gcc编译器学习记录](https://github.com/guodongxiaren/LinuxTool/blob/master/gcc.md)
* [gcc Makefile 入门](http://www.cnblogs.com/jasonliu/archive/2011/12/23/2299740.html)

# 延伸阅读

* [跟我一起写Makefile](http://wiki.ubuntu.com.cn/跟我一起写Makefile)
* [makefile规则](http://blog.sina.com.cn/s/blog_87bbe37e01016f6y.html)