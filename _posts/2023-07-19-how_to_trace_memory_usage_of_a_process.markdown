---
layout: post
title: "获取一个进程的内存最大占用"
date: 2023-07-19 19:26:22 +0800
categories: 笔记
---

在K8S上部署服务的时候，总需要设置内存资源分配。但是对于一个陌生的服务进程，内存占用到底设置成什么样才是合理的呢？

这里有一个办法，那就是在电脑上，实际运行这个服务进程。然后执行期望的操作。当进程退出的时候，最大内存占用就会输出在终端里。

主角就是`time`命令。需要注意的是，`time`本身也是Shell的关键字，直观的执行`/usr/bin/time`更为推荐.

#### MacOS

```
/usr/bin/time -l -h -p <CMD to trigger your process>
```

在输出中，有一行`<regex[0-9]+> peek memory footprint`，就是进程在整个运行周期中使用内存的最大值。正则区匹配的数字就是内存占用，单位为**字节**。

#### Linux

以Debian Bookworm为例。

`time`命令如果没有安装，可以执行

```
sudo apt install -y time
```

安装成功后，

```
/usr/bin/time -f "%M" <CMD to trigger your process>
```

Linux里的`time`命令，使用占位符的方式定制输出项。其中M表示输出进程的内存使用的最大RSS值，单位**KB**。

> RSS (Resident Set Size)，表示进程堆栈的内存占用情况。不会包含共享库以及swap区的占用。
