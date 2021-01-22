---
layout: post
title: Advanced Linux Programming - 设备
date: 2016-05-16 19:53:02
categories: 翻译
---


Linux系统通过模块化的程序——设备驱动，与硬件设备交互。设备驱动将软硬件交互细节隐藏起来，将其封装成标准化的若干操作。

Linux中，设备驱动集成在系统内核中。设备驱动可以静态的连接到内核上，也可以按需动态加载。设备驱动只和内核交互，用户进程无法直接调用设备驱动。Linux系统将设备转化为文件的形式。用户进程对设备的读写操作都和普通文件类似，可以调用低级I/O操作，也可以调用C语言标准库函数。

# 1 设备类型

设备文件和普通文件不一样。它们不占用硬盘空间；对设备文件的读写操作，都是和设备本身的交互。设备文件分为两大类：

* 字符设备文件——这是一种读写字符串流的设备。音频系统、视频系统都是字符设备。
* 块设备文件——表示每次读写固定大小字符块的设备。块设备可以无序地访问设备上的数据，比如硬盘。

通常，用户进程不会直接访问块设备。硬盘分区都挂载到系统根目录树中。用户进程直接访问目录中的文件即可。

>**访问块设备的风险**
>
>Linux系统不允许普通用户进程直接访问块设备。即使具备root权限，进程访问块设备也要倍加小心。稍有不慎，就会改动文件系统的基础信息，例如MBR、分区表等。轻者数据丢失，重者系统崩溃。

# 2 设备号

Linux系统使用两个号码标记设备：主设备号(Major device number)和从设备号(Minor device number)。主设备号决定了设备对应于哪一个驱动程序。这一对应关系是固定的，定义在Linux内核代码中。注意同一个主设备号也许对应于两个不同的驱动程序——一个符号设备和一个块设备。从设备号用来区分，对应于同一个驱动程序的不同设备。从设备号的含义依驱动程序的不同而不同。

主设备号3对应于主IDE控制器。而一个IDE控制器可以控制两个设备：一个“主”设备(Master device)，从设备号为0；一个“从”设备(Slave device)，从设备号为64。“主”设备分区的从设备号，如果可以分区的话，为1，2，3等等；“从”设备分区的从设备号，如果可以分区的话，为65，66，67等等。

在大多数Linux系统中，主设备号的详情可以查阅文档`/usr/src/linux/Documentation/devices.txt`。

# 3 设备文件

设备文件和普通文件很想。可以用`mv`来移动设备文件；也可以用`rm`删除设备文件；如果对设备文件调用`cp`，就可以读取设备；如果覆写设备文件，就相当于向设备写入内容。

要想创建新的设备文件，可以使用shell命令`mknod`(`man 1 mknod`查阅详情)，也可以在C语言中使用`mknod`函数(`man 2 mknod`查阅详情)。仅仅创建设备文件，并不能控制设备。只有root权限进程可以调用`mknod`创建设备文件的同时，创建字符设备或块设备。

root用户终端里，可以使用命令`mknod`创建设备。第一个参数是设备文件的路径；第二个参数为设备类型，`b`代表块设备，`c`代表字符设备；第三、四个参数分别为主设备号、从设备号。

```shell
$ mknod ./lp0 c 6 0
```

使用`ls`命令查看文件时，结果中第一个字符代表文件类型：`-`表示普通文件，`d`表示目录，`b`表示块文件，`c`表示字符文件。如果是后两者，则表示文件大小的位置实际上展示了设备的设备号。

```shell
$ ls -l lp0
crw-r----- 1 root root 6, 0 Mar 7 17:03 lp0
```

使用`stat`命令也可以做到这一点。

## `/dev`目录

通常，Linux系统在`/dev`目录下存放所有设备文件。每一设备文件的名称都和设备号密切相关。

大部分情况下，进程无需创建设备文件。直接使用Linux系统提供的现成设备文件即可。

## 打开文件就可以控制设备

控制设备很简单，就把设备文件当作普通文件打开就可以了。

假设打印机接在电脑的#1并口中，那么要打印文件直接向`/dev/lp0`设备文件传入内容就可以了。

```shell
$ cat document.txt > /dev/lp0
```

在代码中，也可以直接向`/dev/lp0`设备文件传入缓存好的内容。

```C
int fd = open (“/dev/lp0”, O_WRONLY);
write (fd, buffer, buffer_length); 
close (fd);
```

# 4 硬件设备

(略)。原书 P.133-135。

# 5 特殊设备

Linux系统创建了一些特殊文件。这些文件的主文件号都是1，也就是对应于内存驱动程序，而不是一般设备驱动。

## `/dev/null`

`/dev/null`，也就是空设备(null device)，有两个用途：

* Linux系统忽视了所有写入空设备的消息。
* 读取空文件，会立即读到EOF。如果把空设备文件拷贝到另一个文件，这个文件就是一个空文件，字节长度为0。

## `/dev/zero`

`/dev/zero`文件像一个有着无限长度的填充着`0`字节的文件。无论要读取多少字节长度内容，Linux可以返回足够多的`0`。

下面这个hexdump程序可以将文件以十六进制的形式打印出来。

```C
#include <fcntl.h> 
#include <stdio.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	unsigned char buffer[16]; 
	size_t offset = 0;
	size_t bytes_read;
	int i;
	
	/* Open the file for reading. */ 
	int fd = open (argv[1], O_RDONLY);
	
	/* Read from the file, one chunk at a time. 
	 * Continue until read “comes up short”, that is, reads less than we asked for.
	 * This indicates that we’ve hit the end of the file. 
	 */
	do {
		/* Read the next line’s worth of bytes. */
		bytes_read = read (fd, buffer, sizeof (buffer));
		
		/* Print the offset in the file, followed by the bytes themselves. */ 		printf (“0x%06x : “, offset);
		for (i = 0; i < bytes_read; ++i)
			printf (“%02x “, buffer[i]);
		printf (“\n”);
		
		/* Keep count of our position in the file. */ 
		offset += bytes_read;
	}
	
	while (bytes_read == sizeof (buffer));
	
	/* All done. */ 
	close (fd); 
	return 0;
}
```

试着用hexdump读取`/dev/zero`看看。

```shell
$ ./hexdump /dev/zero 

0x000000 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x000010 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x000020 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x000030 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
...
```

当你觉得进程却是停不下来的时候，摁下`Ctrl+C`来终止进程。

使用`/dev/zero`映射内存是比较巧妙的内存声明技巧。

“进程间通信”一章有关`mmap`部分、“Linux系统调用”一章有关`mprotect`部分都提到了这些技巧的使用。

## `/dev/full`

`/dev/full`文件像是一个把整个磁盘空之间都占用了的文件。任何试图写入`/dev/full`文件的操作都会失败，同时`ERRNO`被设置为`ENOSPC`。这一错误代码通常的意思是“目标磁盘空间不足”。

该文件的主要用处是测试程序如何应对“目标磁盘空间不足”这一潜在问题。

## 随机数字设备

`/dev/random`和`/dev/urandom`是Linux系统内置的随机数生成器。

通常的随机数生成办法并不绝对随机。例如C语言函数`rand`，产生的是拟随机数。如果赋予函数相同的seed值，那么总可以得到相同的“随机数”序列。这是因为计算机的特性就是可预测的且结果唯一的 (This behavior is inevitable because computers are intrinsically deterministic and predictable)。

Linux通过测量用户的按键间隔和鼠标移动模式，来生成随机数序列。读取`/dev/random`和`/dev/urandom`就可以获得这样高质量的随机数序列。

这两个个文件的区别是：`/dev/random`的随机资源，也就是用户的按键行为、鼠标移动行为等等，是有限的。如果用完了，就无法继续产生随机数。而`/dev/urandom`不会阻塞，当随机资源使用结束后，该文件会调用算法，将之前产生的随机数列继续拟合生成随机数序列。

所以说，从随机性的角度来讲，`/dev/random`的结果更好。

可以使用`od`命令显示随机文件。`od`命令与之前`hexdump`的区别在于，如果没有字符传入，前者会被阻塞，后者不会被阻塞而直接退出。

```shell
$ od -t x1 /dev/random
...
...
0057300    72  13  fc  2d  c0  e1  75  1c  37  a5  6f  f9  c7  72  69  b0
0057320    ef  82  c6  a2  89  6e  c5  76  20  f7  2c  6f  91  47  9f  c7
0057340    bb  2b  78  14  7c  41  15  28  e4  05  4a  9d  4a  e7  97  ba
0057360    fc  c6  2a  cd  90  42  78  5d  3d  cb  d7  67  07  66  4f  3f
0057400    db  eb  d5  a6  48  84  0c  aa  29  6b  20  6b  a0  e1  68  13
0057420    a7  b3  a0  c7  1f  b3  49  e9  19  a1  74  b4  2b  1f  72  f9
0057440    90  b0  40  5a  5f  93  96  34  9e  90  bd  55  26  ca  35  b9
0057460    89  7c  67  f7  c9  23  c3  07  0e  f1  7e  71  19  e3  34  2a
0057500    d9  d3  e1  74  fc  64  dd  8a  e0  18  da  c3  80  9a  e3  29
0057520    0b  d1  b3  11  90  30  b0  af  2c  12  2b  a8  98  f5  6a  dc
...
...
```

```C
#include <assert.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <fcntl.h> 
#include <unistd.h>

/* Return a random integer between MIN and MAX, inclusive. 
 * Obtain randomness from /dev/random. 
 */
int random_number (int min, int max) 
{
	/* Store a file descriptor opened to /dev/random in a static variable.
	 * That way, we don’t need to open the file every time this function is called. 
	 */
	static int dev_random_fd = -1;
	char* next_random_byte; int bytes_to_read; 
	unsigned random_value;
	
	/* Make sure MAX is greater than MIN. */ 
	assert (max > min);
	
	/* If this is the first time this function is called, 
	 * open a file descriptor to /dev/random. 
	 */
	if (dev_random_fd == -1) {
		dev_random_fd = open (“/dev/random”, O_RDONLY); 
		assert (dev_random_fd != -1);
	}
	
	/* Read enough random bytes to fill an integer variable. */ 
	next_random_byte = (char*) &random_value;
	bytes_to_read = sizeof (random_value);
	
	/* Loop until we’ve read enough bytes. 
	 * Because /dev/random is filled from user-generated actions, 
	 * the read may block and may only
	 * return a single random byte at a time. 
	 */ 
	 do {
		int bytes_read;
		bytes_read = read (dev_random_fd, next_random_byte, bytes_to_read); 		bytes_to_read -= bytes_read;
		next_random_byte += bytes_read;
	} while (bytes_to_read > 0);

	/* Compute a random number in the correct range. */
	return min + (random_value % (max - min + 1));
}
```

## 回环设备

回环设备允许使用普通磁盘文件来模拟块设备。这个磁盘文件必须大于要模拟的磁盘。

回环设备文件路径名类似于`/dev/loop0`，`/dev/loop1`等等。每一个文件每次只能模拟一个块设备。只有root用户才可以设置回环设备。

回环设备文件可以用来设置虚拟文件系统。也就是在一个普通磁盘文件中构建独立的文件系统，然后像正常的磁盘、分区一样挂载到系统目录中。

如何创建虚拟文件系统呢？

1. 新建一个空文件。文件大小就是挂载后的设备大小。一个好方法是从`/dev/zero`空文件中使用`dd`命令，逐块拷贝。
	
	```shell
	$ dd if=/dev/zero of=/tmp/disk-image count=20480 
	20480+0 records in
	20480+0 records out
	
	$ ls -l /tmp/disk-image
	-rw-rw---- 1 root root 10485760 Mar 8 01:56 /tmp/disk-image
	```

2. 这个文件暂时里面填充的都是`0`字节。挂载之前必须先格式化。使用``命令将虚拟磁盘格式化为ext2格式。

	```shell
	$ mke2fs -q /tmp/disk-image
	
	mke2fs 1.18, 11-Nov-1999 for EXT2 FS 0.5b, 95/08/09 
	disk-image is not a block special device.
	Proceed anyway? (y,n) y
	```
3. 使用回环设备文件挂载普通文件。
	首先要设置挂在点`/tmp/virtual-fs`。

	```shell
	$ mkdir /tmp/virtual-fs
	$ mount -o loop=/dev/loop0 /tmp/disk-image /tmp/virtual-fs
	```
	
4. 然后切换工作路径到挂在点，就进入了虚拟磁盘。
5. 当不再使用虚拟磁盘时，可以卸载虚拟磁盘文件。
	
	```shell
	$ cd /tmp
	$ umount /tmp/virtual-fs
	```

# 6 伪终端PTY

如果在终端里不加参数，直接运行`mount`命令会返回当前系统加载的文件系统。其中有一行像这样

```shell
none on /dev/pts type devpts (rw,gid=5,mode=620)
```

这是一种特殊的文件系统`devpts`，挂载在`/dev/pts`目录下。这一文件系统和任何硬件设备都没有关系。`/dev/pts`目录里的文件也是设备文件，但是由Linux自动生成的，其内容也随系统运行不断变化。

`/dev/pts`文件对应于伪终端(pseudo-terminals，简写为PTY)。Linux给每一个终端窗口都建立一个PTY，并将每条命令以及返回结果存入`/dev/pts`文件。PTY设备的表现很像终端：从键盘接受输入，并显示程序运行结果。PTY是有序号的，它的序号就是`/dev/pts`里相应的条目名称。

使用`ps -o tty,pid,cmd`命令可以查看当前使用的PTY。

```shell
$ ps -o pid,tty,cmd

 PID  TT    CMD
28832 pts/4 bash
29287 pts/4 ps -o pid,tty,cmd

$ ls -l /dev/pts/4

crw--w---- 1 samuel tty 136, 4 Mar 8 02:56 /dev/pts/4

```

向PTY文件传入字符，也会显示在终端窗口里。假设新建一个终端窗口对应PTY为`/dev/pts/7`。

```shell
$ echo ‘Hello, other window!’ > /dev/pts/7
```

# 7 `ioctl`调用

`ioctl`调用可以用来控制硬件设备。其第一个参数是要控制的设备文件描述符，第二个参数是要执行的操作。虽操作不同，还可能又需要传入更多参数。操作符定义可以在Linux手册`ioctl_list`项中查找。

下面代码cdrom-eject.c展示了光驱弹出的实现。

```C
#include <fcntl.h> 
#include <linux/cdrom.h> 
#include <sys/ioctl.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	/* Open a file descriptor to the device specified on the command line. */ 
	int fd = open (argv[1], O_RDONLY);
	/* Eject the CD-ROM. */
	ioctl (fd, CDROMEJECT);
	/* Close the file descriptor. */ 
	close (fd);

	return 0;
}
```

如果计算机的第二IDE控制器的主设备是一个CD-ROM，可以使用该程序弹出光驱：

```shell
$ ./cdrom-eject /dev/hdc
```

