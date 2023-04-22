---
layout: post
title: "OpenRC使用小结"
date: 2023-04-22 14:20:29 +0800
categories: 笔记
---

OpenRC是为类Unix系统设计的进程管理系统，可以在系统的启动和关闭时候自动管理后台服务。

OpenRC是Gentoo社区开发维护的。除了Gentoo Linux之外，Alpine Linux也默认使用OpenRC做进程管理。除了Linux外，OpenRC也可用于BSD系统，例如[GhostBSD](https://wiki.ghostbsd.org/index.php/OpenRC)。

尽管OpenRC的受众看上去偏少数，但在Slant所作的一份[在线调查《What are the best Linux init systems?》](https://www.slant.co/topics/4663/~linux-init-systems)中，OpenRC名列第一。而主流Linux使用的SystemD未能进入前5名。

我个人的感觉是OpenRC的设计确实很不错：简单易上手，功能也不差，符合Unix的KISS哲学。

## 服务配置文件

以Alpine Linux为例，OpenRC的服务自定义配置放置在`/etc/init.d/`目录下，配置文件名就是服务的名字。

OpenRC配置文件的语法类似于Shell脚本，因此学习成本很低。OpenRC配置至少需要声明`start()`, `stop()`, `status()`三个函数。

`rc-service`命令就是执行服务配置文件中对应的函数。假设我们有一个`some_service`服务，为了启动该服务，我们可以执行命令`rc-service some_service start`。其本质就是在执行`/etc/init.d/some_service`中的`start()`函数。为了方便配置文件的编写，OpenRC还提供了一个默认的配置文件。自定义配置可以调用默认配置的函数，甚至可以直接继承默认配置的函数。

假设，我们的自定义服务，在启动后并没有后台进程，只是想在系统启动时执行若干命令。那我们可以自定义`start()`函数，在调用期望的命令后，执行默认函数的`default_start`函数。那我们就无需再定义`status()`函数。直接利用默认配置的行为来确认是否时候启动完成就好了。

```
## /etc/init.d/some_service
## exmaple of OpenRC service file

start()
{
    # execute necessary commands
    ...

    default_start
}
```

当服务启动完成后，我们可以使用命令查看其状态

```
$ rc-service some_service status
* status: started
```

或者

```
$ rc-status
...
Runlevel: default
...
some_service              [started]
...
```

此外，OpenRC还支持用户自定义服务的命令函数。详见：[Syntax of Service Scripts](https://github.com/OpenRC/openrc/blob/master/service-script-guide.md#syntax-of-service-scripts)

### 开机自动启动服务

`rc-service`只能执行服务的命令。如果需要让服务能够开机自动启动，则需要执行`rc-update`命令。

```
rc-update add some_service default
```

在systemd中，如果自定义服务的启动配置中，如果有依赖的服务，需要在配置文件中一一列出。如果其中的依赖服务发生变动，其他相关的依赖项都需要调整。

OpenRC则很好的将这个问题简单化了。OpenRC引入了一个概念，叫runlevel，即运行级别。OpenRC会按顺序处理不同运行级别的服务，同一个级别的服务则根据字母顺序启动。一般系统层面的服务，例如网络服务，都会在`sysinit`和`boot`两个级别内。而用户层面的服务放在`default`级别中。这样一来，用户服务无需再声明系统服务的依赖项等。

前面这条命令，就是将服务`some_service`添加至runlevel `default`中。

### 延伸阅读

- [OpenRC - Gentoo Wiki](https://wiki.gentoo.org/wiki/OpenRC)
- [OpenRC Users Guide](https://github.com/OpenRC/openrc/blob/master/user-guide.md)
- [OpenRC Service Script Writing Guide](https://github.com/OpenRC/openrc/blob/master/service-script-guide.md)
