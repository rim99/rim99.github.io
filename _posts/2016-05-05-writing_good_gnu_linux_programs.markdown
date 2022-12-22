---
layout: post
title: Advanced Linux Programming - 编写出色的GNU/Linux程序
date: 2016-05-05 19:53:02
categories: 翻译
---

><http://advancedlinuxprogramming.com>提供了本书电子版的免费下载。

# 1 与执行环境交互
### 关于参数

C语言程序的main()函数使用两个参数和执行环境交互——`(int)argc`和`(char*)argv[]`。前者表示所执行命令的参数个数，后者包含了所执行命令的各个参数。`(char*)argv[]`的第一个元素为程序本身的位置，其后为程序的参数。

看例子：

```C
#include <stdio.h>

int main(int argc, char* argv[])
{
	printf("The name of the program is "%s".\n", argv[0]);
	printf("The program was invoked with %d arguments.\n", argc);
	
	if (argc > 1) {
		printf("The argments are:\n");
		for (int i = 1; i < argc; i++) {
			printf("   %s\n", argv[i]);
		}
	}
	return 0;
}
```

程序参数有两种：

* 短参数－只有一个连字符、后面通常只有一个小写字母，如“-s”
* 长参数－有两个连字符、后面通常有多个大小写字母组成的字符，如“--size”

通常，Linux程序的每一个短参数都有一个对应的长参数。

Linux对于命令行参数的选择有一份指南[GNU Coding Standards](http://www.gnu.org/prep/standards/standards.html)。

Linux在`getopt.h`头文件中提供了`getopt_long`函数可以专门处理参数列表，**建议使用**。原书程序示例见 P.21－23，也可以用`man getopt_long`查看官方文档。

## 标准I/O

Linux有三类标准流文件，stdin，stdout，stderr。三者在Linux中也可以用文件描述符来表示：0，1，2。通常程序的错误信息最好输出到stderr中，以免与标准输出信息相混淆。

```C
fprintf(stderr, "Error:(...)");
```

值得注意的是，stdout是有缓存的。只有缓存填满、程序正常退出或者stdout关闭时，才会真正输出。Linux中可以使用`fflush()`函数来强制清洗缓存，具体看Linux手册。

而stderr没有缓存，可以即时输出。

在Shell中可以使用“2>&1”，来实现stderr与stdout的合并。例如，

```shell
$ program > out_put.file 2>&1
$ program > 2>&1 | filter
```

## 程序退出代码

程序正常退出时，通常代码为0。如果遇到非零代码，则表示程序出错。

Shell中使用`$?`表示最近执行程序的退出代码。

```shell
$ ls -s /

total 64
4 bin   0 dev  4 home  4 lib64  4 mnt  0 proc  4 run   4 srv  4 tmp  4 var
4 boot  4 etc  4 lib   4 media  4 opt  4 root  4 sbin  0 sys  4 usr

$echo $?

0
```

## 环境变量

Linux使用大写字母表示环境变量。例如`USER`表示用户名，`HOME`表示home目录位置。在Shell中，可以使用`export`来改变环境变量：`export USER=your_name`。在C语言中，使用`stdlib.h`定义的`setenv`和`getenv`函数来获取或者修改环境变量。

## 使用临时文件

程序运行时往往需要新建临时文件来存储数据或传输数据。在Linux系统中，临时文件存放在`/tmp`目录下。使用临时文件需要注意以下几点：

1. 同一程序往往会同时运行，可能是同一用户所为，也可能是不同的用户。应当确保这些程序实例拥有不同的临时文件目录，以免发生冲突。
2. 临时文件的权限应当设定为未授权用户不得修改或替换临时文件。
3. 临时文件的文件名不能有固定模式，不应当被预测到；否则会被恶意程序利用。

Linux提供了`mkstemp`函数和`tmpfile`函数来处理临时文件。原书 P.28-29 展示了`mkstemp`函数的使用案例。


# 2 防御性的代码

## 使用`assert`

`assert`主要用来检查运行时错误，其参数为一个boolean表达式。如果表达式为假，就会打印错误信息。

`assert`宏会大幅降低程序性能。这个可以在头文件中使用`NDEBUG`宏来编译规避。

```C
int func(argv);

void test1() {
	assert(func(argv1)==0);
	assert(func(argv2)==0);
}

void test2() {
	int status = func(argv3);
	assert(status == 0);
} 

```

最好在下列情况下使用`assert`宏：

1. 检查指针是否为空。`assert(pointer != NULL)`所产生的错误信息是这样的——`Assertion "pointer != ((void *)0)" failed.`，相比于普通运行时产生的`Segmentation fault (core dumped)`来说要有用的多。
2. 检查函数参数是否合法。比如函数`foo()`的参数`arg`只能为正，可以在`foo()`函数体内添加表达式`assert(arg > 0)`。从而可以检测出函数是否会错误的使用，也可以提高代码的可读性。

## 系统调用失败

程序的系统调用往往会在下列情况中失效：

1. 系统所分配的资源耗尽。比如程序索取很多内存空间，向磁盘中写入内容太多，或同时打开太多文件。
2. 当程序要求权限之外的系统调用时，Linux会屏蔽该调用。
3. 如果用户提供了非法参数或者程序本身有bug，系统调用都会失败。
4. 程序之外的原因，例如硬件问题，也会导致系统调用失败。
5. 系统调用也会被诸如信号中断之类的事件所打断。这种情况下，重启系统调用即可。

## 系统调用的错误代码

大多数情况下，系统调用会返回0值表示成功，返回非零值表示出错。当然也有例外，比如`malloc`正常的会返回非零值表示指针地址，返回NULL指针则表示出错。因此，应该仔细查看Linux手册，确认系统调用的定义。

大多数系统调用会在失败时向变量`errno`中存入更多的错误信息。每一个系统调用失败都会覆写变量`errno`。因此，一旦发生失败，就应该立即转存`errno`中的信息。这些错误信息都是整数，其可能值都定义在预处理器中。如果要使用变量`errno`，应当引入`<error.h>`文件。、、

Linux在`<string.h>`提供更方便的函数`strerror`。该函数返回一条字符形式的`errno`中的错误信息。Linux还在`<stdio.h>`提供函数`perror`，能够直接将错误输出到stderr流中。

下列代码展示了`strerror`的用法：当文件打开失败时向stderr输出错误信息。

```C
fd = open("text.txt",O_RDONLY);
if (fd == -1) {
	/* 打开失败 */
	fprintf(stderr, "Error opening file: %s\n", strerror(errno));
	return -1;
}
```

一个比较特殊的错误信息是`EINTR`。在诸如`read`、`select`、`sleep`这样的阻塞函数运行时，如果程序接收到中断请求，就会停止这些函数的运行，同时`errno`设定为`EINTR`。

P.34页代码展示了如何使用`case-switch`结构处理`errno`值。

## 处理系统调用失败

通常，程序遇到系统调用失败后，最好先停止当前的任务，但不要终止程序。因为所遇到的错误也许是可恢复的。一个可行的办法是，从当前函数返回，并传递返回码给调用者指出错误所在。如果在出错之前已经分配了资源，那么这些资源应当释放掉，以免发生内存泄漏。

举个例子，一个函数要将文件内容读取到缓存中，需要下列几步：

1. 分配缓存空间；
2. 打开文件；
3. 读取文件到缓存；
4. 关闭文件；
5. 返回缓存指针。

如果文件不存在，第2步就会失败，函数多返回返回NULL指针。但是此时缓存空间已经分配好了，如果不回收就会发生内存泄漏。如果第3步失败了，不但要回收缓存空间，还要关闭文件，释放描述符。原书P.35-36展示了相关代码。

# 3 编写并使用库函数文件

程序的运行离不开库函数文件。程序可以静态地或者动态地链接到库函数文件。静态链接会导致程序更大，更难升级，但容易部署；动态链接则相反。

## 静态库

静态库，或者叫归档(archive)，就是把一组对象文件合起来存储在同一个文件中。当把代码和归档文件链接起来时，代码会在归档文件中搜索所需要的对象文件，解压缩，然后将对象文件与代码链接起来。归档文件通常使用.a后缀名，对象文件则是.o后缀名。下面代码将test1.o、test2.o两个对象文件归档在libtest.a中。

```shell
$ ar cr libtest.a test1.o test2.o # cr告诉ar创建归档文件
```

## 共享库

共享库，又叫共享对象、动态链接库，和静态库类似，也是一组对象文件的集合，以.so为后缀名。但不同的是，当共享库链接到代码中编译成为可执行文件时，其中并不包含共享库中的代码，而仅仅包含一指向共享库的索引(refrence)。共享库也不包含所有代码，而仅仅包含所需要的代码。

要创建共享库文件，得先编译出对象文件：

```shell
$ gcc -c -fPIC test1.c
```

然后把对象文件合并为共享库文件：

```shell
$ gcc -shared -fPIC -o libtest.so test1.o test2.o
```

程序在运行时，会在`/lib`、`/usr/lib`两个文件夹里搜索共享库。如果共享库文件不在这两个位置，就会出错。一个解决办法是：在gcc中使用`-Wl,-rpath,/dir`选项将`/dir`目录设定为搜索目录。另一个办法是：设置`LD_LIBRARY_PATH`环境变量。例如`echo LD_LIBRARY_PATH=/dir1:/dir2`，那么`/dir1`和`/dir2`两个目录就会在两个标准目录之前搜索。注意，目录之间用冒号`:`分割。

另外参考：[第一章 C语言 —— 静态库和动态库](http://rtxbc.iteye.com/blog/1132114)。

## 标准库

即使没有任何显式的指明，gcc仍然会将所有程序与C语言的标准库libc文件相链接。如果代码中用到了数学计算函数，就应该用`-lm`选项链接C语言的数学标准库文件libm。

## 动态加卸载库文件

`<dlfcn.h>`头文件中包含了一组函数允许程序在运行过程中动态地加载或卸载库文件。

```C
// 打开库文件
void* handle = dlopen("libtest.so", RD_LAZY); 

// 新建函数指针指向库函数
void (*test)() = dlsym(handle, "my_func");  

// 运行库函数
(*test)(); 

// 关闭库文件
dlclose(handle);  
```









