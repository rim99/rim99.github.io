---
layout: post
title: Advanced Linux Programming - 进程
date: 2016-05-12 19:53:02
categories: 翻译
---

程序的运行着的实例叫做进程。Unix中，大多数有关进程管理的函数都声明在`<unistd.h>`头文件中。

# 1 进程初步

## 进程ID

Linux中每一个进程都一个独立的ID标识，即pid。进程ID随着进程产生时由系统分配，是一个16位的数字。Linux中，出了特殊的init进程外，每一个进程都有一个父进程。父进程ID叫做ppid。

C语言中进程ID为`pid_t`类型，定义在`<sys/types.h>`文件中。在进程中执行`geipid()`和`getppid()`可以分别获去当前进程的pid和ppid。

## 查看运行中的进程

Shell中使用`ps`命令可以查看当前运行的进程。

```shell
$ ps -e -o pid,ppid,command
  
  PID  PPID COMMAND
    1     0 /sbin/launchd
   44     1 /usr/sbin/syslogd
   45     1 /usr/libexec/UserEventAgent (System)
   47     1 /usr/libexec/kextd
  ...
```

`-e`选型表示列出所有在运行进程，包括后台系统进程。`-o pid,ppid,command`表示需要输出进程ID，父进程ID，以及进程启动命令三项。

# 2 创建进程

## `system`函数

`system`函数定义在`<stdlib.h>`文件中。该函数直接调用Shell——`/bin/sh`来执行命令。如果Shell本身无法调用，`system`函数返回`127`值；如果遇到其他错误，就返回`-1`。

```C
#include <stdlib.h>

int main() {
	int return_value;
	return_value = system("ls -l /");
	return return_value;
}
```

`system`函数较为危险。因为他调用的是`/bin/sh`。而`/bin/sh`在不通的Linux中指向不同的的Shell，有些是bash，有些是zsh，也可能碰到tcsh。而且不同的Linux发型版赋予`system`函数的权限也不完全相同。

因此，不推荐使用`system`函数创建进程。

## `fork`函数和`exec`族函数

Linux中，`fork`函数创建当前进程的一个独立拷贝，`exec`函数则保证新进程的内容与父进程无关。

### 调用`fork`函数

程序调用`fork`函数后会复制当前进程，产生出一个子进程。父进程和子进程都会从“fork”处持续执行代码。两者的仅有的区别就是pid和ppid。

执行`fork`函数后会出现两个进程，有不同的返回值。在父进程中，`fork`函数的返回值是子进程的pid；子进程中，`fork`函数的返回值是`0`。

```C
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main() 
{
	pid_t child_pid;
	printf("The main process ID is : %d.\n", (int) getpid()); 
	
	child_pid = fork();
	
	if (child_pid != 0) {
		/* parent process */
		printf("This is a parent process with ID: %d.\n", (int) getpid());
		printf("The child process ID is: %D.\n", (int) child_pid);
	}else{
		/* child process */
		printf("This is a child process with ID: %d.\n", (int) getpid());
	}
	
	return 0;
}
```


### 调用`exec`族函数

`exec`族函数可以将当前进程中运行的程序替换为其他程序。当程序调用`exec`族函数后，当前进程立即终止作业，从头执行新程序——根据`exec`族函数的参数。

`exec`族函数是这样的：

* 名字中有字母“p”的`execvp`、`execlp`接受一个程序名称，并在当前执行路径中搜索程序。不含“p”的函数必须给出程序的全部路径。
* 名字中有字母“v”的`execv`、`execvp`、`execve`可以将一组以`NULL`结尾的、字符串指针数组作为新程序的参数列表。名字中有字母“l”的`execl`、`execlp`、`execle`函数使用C语言的`varargs`机制作为参数列表。
* 名字中有字母“e”的`execve`、`execle`可以接受额外的参数——一组环境变量。该数组为以`NULL`结尾的、字符串指针数组。每一个字符串的形式应为：“VARIABLE＝value”。

除非发生错误，否则`exec`族函数不会返回任何值。

### 同时调用

通常，先用`fork`函数，生成一个子进程，然后再子进程中执行`exec`族函数。这样父进程可以继续当前作业，不受打扰。

```C
#inlcude <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

int spawn(char* program, char** arg_list)
{
	pid_t child_pid;
	child_pid = fork();
	
	if (child_pid != 0) {
		/* parent process */
		return child_pid;
	}else{
		/* child proces */
		execvp(program, arg_list);
		fprintf(stderr, "Error occcured in excevp.\n");
		abort();
	}
}

int main()
{
	char* arg_list = {
		"ls",
		"-l",
		"/",
		NULL
	};
	
	spawn("ls", arg_list);
	printf("Done with main program.\n");
	
	return 0;
}
```

## 进程调度

Linux不能保证父子进程的先后顺序，只能保证每一个都能够得到执行。如果进程间有优先级，那么需要设定`niceness`参数。

在Shell中，使用`nice`程序设定命令的优先级。看例子：

```Shell
$ nice -n 10 ls -l /
```

这条命令将命令`ls -l /`的`niceness`参数设定为10。`niceness`数值越高，进程的优先级越低，所能得到的执行时间越少。

对于正在执行的进程，可以使用`renice`命令来改变其优先级。

需要注意的是，只有root用户才可以把`niceness`参数设定为负值。

# 3 信号

信号是发给进程的特殊消息。信号是异步的：当程序接收到信号，会立即暂停当天作业，改为处理信号。信号有很多种，根据信号代码来区别。信号代码定义在`/usr/include/bits/signum.h`文件中。C语言中直接引用`<signal.h>`头文件即可。

进程接收到信号后，可以根据信号的对策(disposition)，做出不同的反应。每一种信号都一个默认对策(default-disposition)，如果进程没有选择其他对策，系统就会执行其默认对策。对于大多数信号，进程可以选择忽略，或者调用“信号句柄(Singnal-Handler)”函数来处理。如果要使用信号句柄函数，当前作业就会暂停。当信号句柄函数返回后，当前作业继续执行。

当进程试图执行非法操作时，Linux会向进程发送诸如SIGBUS(总线错误，bus error)，SIGSEGV(段违规，segmentation vialation)，SIGFPE(浮点异常，floating point exception)等信号，并试图终止进程。

使用`sigaction`函数来设置信号的默认对策。第一个参数是信号代码，后两个是指向sigaction结构的指针：第一个包括信号的默认对策，第二个包含信号以前的对策，(The first of these contains the desired disposition for that signal number, while the second receives the previous disposition)。 sigaction结构中最重要的`sa_handler`域可以是下面三个中的某一个值：

* SIG_DFL，定义信号的默认对策；
* SIG_IGN，定义信号是否可忽略；
* 一个信号句柄函数指针。该函数只接受一个参数，信号代码，返回void。

由于信号的处理是异步的，当信号句柄函数处理信号时，主程序处于一个非常脆弱的状态。因此，信号句柄函数中因当避免I/O操作，或对大多数库、系统函数的调用。信号句柄函数因当根据信号执行尽可能少的作业量，然后将控制权返还主程序。主程序会周期性的检查是否有信号到达，并执行相应的作业。尽管不常见，信号句柄函数还是能够因为其他信号而被暂停，注意由此带来的不便。

即使是修改全局变量也会很危险。试考虑，当全局变量值的修改需要执行两步机器指令时，如果又一个同类信号发生在这两步之间，再次试图对全局变量作出修改，此时前一信号句柄函数执行第二步机器指令时很可能会出错。

如果信号句柄函数要使用全局变量标记信号，那么这个变量最好是`sig_atomic_t`类型。Linux保证该类型的修改只需要一条机器指令即可完成。实际上Linux中，`sig_atomic_t`类型就是`int`类型。对于`int`类型、指针类型或者更小的类型，其修改数值的操作都是原子性的。

```C
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

sig_atomic_t sigusr1_count = 0;

void handler (int signal_num)
{
	++sigusr1_count;
}

int main() 
{
	struct sigaction sa;
	memset(&sa, 0, sizeof(sa));
	sa.sa_handler = &handler;
	sigaction(SIGUSR1, &sa, NULL);
	
	/* Do some lengthy stuff here. */
	/* ... */
	
	printf(“SIGUSR1 was raised %d times\n”, sigusr1_count);
	
	return 0;
}
```

# 4 进程中止

进程中止的方法有很多。

程序可以调用`exit`函数，也可以在main函数中返回来正常退出。`exit`函数的参数或者main函数的返回值就是进程的退出代码(exit code)。退出代码的规范可见上一篇笔记[《编写出色的GNU/Linux程序》](http://www.rim99.com/blogpost/writing_good_gnu_linux_programs)。

进程也会因为收到各类信号而异常退出。例如前面提到的SIGBUS，SIGSEGV，SIGFPE等信号。此外，如果用户按下Ctrl+C，进程会收到SIGINT信号；Shell中的kill命令会发出SIGTERM信号，两者的默认对策都是终止进程。通过调用`abort()`函数，进程想自己发出SIGABRT信号来中止自身，并生成一个核心文件(core file)。最强大的中止信号是SIGKILL，该信号能够立即无条件中止进程，无法被阻塞或应对(handled)。

这些信号可以在Shell中用kill命令发出：

```Shell
$ kill -KILL <pid>
```
也可以在程序中用`kill`函数来发送：

```C
#include <sys/types.h>
#include <signal.h>

kill(child_pid, SIGTERM);
```

## 等候进程中止

对于使用`fork`函数和`exec`族函数产生子进程的情形来说，通常父进程之行结束后，子进程才会产生结果。

如果想要父进程等待子进程结束后再执行，可以使用`wait`族信号。

## `wait`系统调用

最简单的函数就是`wait`，能够阻塞父进程直到其某一子进程退出或出错。该函数通过整数指针参数返回退出代码。`WEXITSTATUS`宏包含了子进程的退出代码。`WIFEXITED`宏可以根据子进程退出代码判定进程是正常退出还是被信号中止。对于后者，可以使用`WTERMSIG`宏来确认是被什么信号中止的。

使用`waitpid`函数可以指定等待的进程；`wait3`函数可以返回进程的CPU使用统计；`wait4`函数可以对指定等待的进程提供更多选择。

## 僵尸进程

僵尸进程(zombie process)就是一条不再运行但其资源仍然未被回收的进程。考虑父进程fork出子进程后调用了wait函数。如果在父进程调用wait之前，子进程还没中止。那么父进程被阻塞直到子进程中止。如果子进程已经中止了，那么在父进程调用wait之前，该子进程即使一条僵尸进程。父进程调用wait之后，子进程被回收。

如果父进程结束后，子进程才结束。那么子进程将永远成为僵尸进程。

## 异步清除子进程

如果父进程fork出多条子进程，而且父进程不能被阻塞。那么子进程的回收就成了问题。

一个解决办法是周期性的调用`wait3`或`wait4`函数。如果向这两个函数传入`WNOHANG`，函数就会运行在非阻塞模式：如果有僵尸子进程，回收并返回其pid；如果没有，返回0。

这里有一个更优雅的解决方案。Linux会在子进程中止时向父进程发送SIGCHLD信号，其默认对策是什么都不做。因此可以修改父进程设定——每次收到SIGCHLD信号就调用一次wait来回收资源。

```C
#include <signal.h> 
#include <string.h> 
#include <sys/types.h> 
#include <sys/wait.h>

sig_atomic_t child_exit_status;

void clean_up_child_process (int signal_number) {
	/* Clean up the child process. */
	int status;
	wait(&status);
	/* Store its exit status in a global variable. */ 
	child_exit_status = status;
}

int main () {
	/* Handle SIGCHLD by calling clean_up_child_process. */
	struct sigaction sigchld_action;
	memset(&sigchld_action, 0, sizeof (sigchld_action));
	sigchld_action.sa_handler = &clean_up_child_process;
	sigaction(SIGCHLD, &sigchld_action, NULL);

	/* Now do things, including forking a child process. */ 
	/* ... */	
	
	return 0;
}
```






































