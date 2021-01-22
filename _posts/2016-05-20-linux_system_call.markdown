---
layout: post
title: Advanced Linux Programming - Linux系统调用
date: 2016-05-20 19:53:02 +8000
categories: 翻译
---
 

在前面，我们接触到了很多函数能够实现系统相关的功能，比如解析命令行参数、控制进程以及映射内存等等。实际上，这些函数能够分为两大类：

* **库函数**——这些函数就像普通函数一样，参数放置在寄存器或者栈里，运行时就从动态库里加载。

* **系统调用**——这类函数的参数被打包传递到内核，由内核执行作业。例如低级I/O操作，`open`或者`read`。

Linux提供了200多种不同的系统调用。他们大多声明在`/usr/include/asm/unistd.h`文件里。


# 1 `strace`命令

`strace`命令能够跟踪另一个程序的执行情况，给出其执行的系统调用和接收到的信号中断。

如果要追踪`hostname`命令，就可以执行：

```shell
$ strace hostname

execve("/bin/hostname", ["hostname"], [/* 10 vars */]) = 0
brk(0)                                  = 0x255f000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe7b3f38000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=16300, ...}) = 0
mmap(NULL, 16300, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fe7b3f34000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libnsl.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0`A\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0644, st_size=97296, ...}) = 0
mmap(NULL, 2202328, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fe7b3afe000
mprotect(0x7fe7b3b15000, 2093056, PROT_NONE) = 0
mmap(0x7fe7b3d14000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16000) = 0x7fe7b3d14000
mmap(0x7fe7b3d16000, 6872, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fe7b3d16000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\320\37\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1840928, ...}) = 0
mmap(NULL, 3949248, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fe7b3739000
mprotect(0x7fe7b38f4000, 2093056, PROT_NONE) = 0
mmap(0x7fe7b3af3000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1ba000) = 0x7fe7b3af3000
mmap(0x7fe7b3af9000, 17088, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fe7b3af9000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe7b3f33000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe7b3f31000
arch_prctl(ARCH_SET_FS, 0x7fe7b3f31740) = 0
mprotect(0x7fe7b3af3000, 16384, PROT_READ) = 0
mprotect(0x7fe7b3d14000, 4096, PROT_READ) = 0
mprotect(0x602000, 4096, PROT_READ)     = 0
mprotect(0x7fe7b3f3a000, 4096, PROT_READ) = 0
munmap(0x7fe7b3f34000, 16300)           = 0
brk(0)                                  = 0x255f000
brk(0x2580000)                          = 0x2580000
uname({sys="Linux", node="391e4e96744b", ...}) = 0
fstat(1, {st_mode=S_IFCHR|0600, st_rdev=makedev(136, 0), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe7b3f37000
write(1, "391e4e96744b\n", 13391e4e96744b
)          = 13
exit_group(0)                           = ?
+++ exited with 0 +++
```

随后的输出结果中每一行都代表一个系统调用——系统调用名称、参数以及返回值。`strace`命令只能显示系统调用，不能显示普通函数。

`strace`命令有助于记录程序的运行过程。

# 2 `access`－测试文件权限

`access`可以检查调用进程是否有文件的读写或执行权限，也可以用来检查文件是否存在。

`access`接收两个参数：文件路径；测试权限：`R_OK`、`W_OK`、`X_OK`分别对应于读、写、执行等权限。如果测试权限全部通过，调用返回0；如果文件存在，但是权限测试没有通过，就会返回-1，并将ERRNO设置为EACCES（或者EROFS，如果是在一个只读文件上测试写权限）。

如果第二个参数设置为`F_OK`，调用就只会测试文件是否存在。如果文件存在，返回0；不过不存在，返回-1，并将ERRNO设置为`ENOENT`。如果文件路径中的某一级目录都无法访问，那么ERRNO也会被设置为`EACCES`。

```C
#include <errno.h> 
#include <stdio.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	char* path = argv[1]; 
	int rval;
	
	/* Check file existence. */ 
	rval = access (path, F_OK); 
	if (rval == 0)
		printf (“%s exists\n”, path); 
	else {
		if (errno == ENOENT)
			printf (“%s does not exist\n”, path);
		else if (errno == EACCES)
			printf (“%s is not accessible\n”, path);
		return 0; 
	}
	
	/* Check read access. */ 
	rval = access (path, R_OK); 
	if (rval == 0)
		printf (“%s is readable\n”, path); 
	else
		printf (“%s is not readable (access denied)\n”, path);
	
	/* Check write access. */ 
	rval = access (path, W_OK); 
	if (rval == 0)
		printf (“%s is writable\n”, path); 
	else if (errno == EACCES)
		printf (“%s is not writable (access denied)\n”, path); 
	else if (errno == EROFS)
		printf (“%s is not writable (read-only filesystem)\n”, path); 
	return 0;
}
```

# 3 `fcntl`

`fcntl`调用能够对打开的文件描述符执行高级操作。第一个参数是文件描述符；第二个参数指明了执行哪一种操作。这里解释一下锁定文件的操作。其他用途可以查看man手册。

`fcntl`调用加锁类似于多进程中的互斥锁。可以有多个拥有读取锁的进程同时读取文件，但所有拥有写入锁的进程中，只能同时有一个进程写入文件。需要注意的是，没有调用`fcntl`的进程可以无限制读写文件。也就是，`fcntl`调用的加锁只适用于同样调用`fcntl`的进程。

要锁定文件，得先新建一个`flock`结构变量。将其中的`l_type`域设定为`F_RDLCK`或`F_WRLCK`，分别对应于读取锁、写入锁。然后调用`fcntl`，传入文件描述符、`F_SETLCKW`操作符以及前面结构体的指针。如果已经有进程持有锁，那么新的`fcntl`调用就会保持阻塞直到前一个锁被释放。

下面代码可以将命令行里的文件加上写入锁，然后等待用户按下ENTER键，再执行解锁并关闭文件。

```C
#include <fcntl.h> 
#include <stdio.h> 
#include <string.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	char* file = argv[1]; 
	int fd;
	struct flock lock;
	printf (“opening %s\n”, file);
	
	/* Open a file descriptor to the file. */ 
	fd = open (file, O_WRONLY);
	printf (“locking\n”);
	
	/* Initialize the flock structure. */ 
	memset (&lock, 0, sizeof(lock)); 
	lock.l_type = F_WRLCK;
	
	/* Place a write lock on the file. */ 
	fcntl (fd, F_SETLKW, &lock);
	printf (“locked; hit Enter to unlock... “); 
	
	/* Wait for the user to hit Enter. */ getchar ();
	printf (“unlocking\n”);
	
	/* Release the lock. */ 
	lock.l_type = F_UNLCK;
	fcntl (fd, F_SETLKW, &lock);
	close (fd);
	return 0;
}
```

如果想让`fcntl`执行非阻塞加锁操作，可以把`F_SETLKW`替换为`F_SETLK`。如果无法加锁，就会立即返回-1。

Linux里`flock`也可以执行相似的文件加解锁操作，但是`fcntl`的优势是可以在NFS文件系统中使用。

# 4 `fsync`和`fdatasync`——冲刷磁盘缓存

Linux系统通常会将进程的写磁盘操作暂存在磁盘缓存上，等到缓存满了或者其他条件满足时，将所有内容一并写入。但是如果，系统突然崩溃或者系统电源失效，缓存中的内容会立刻消失无法挽回。所以，有些重要内容需要立即写入磁盘，而不是留在缓存。

Linux的`fsync`调用可以将文件立刻写入磁盘。该调用只需要一个参数——文件描述符。`fsync`会一直阻塞直到所有数据写入磁盘。

```C
#include <fcntl.h> 
#include <string.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

const char* journal_filename = “journal.log”;
void write_journal_entry (char* entry) 
{
	int fd = open (journal_filename, O_WRONLY | O_CREAT | O_APPEND, 0660); 
	write (fd, entry, strlen (entry));
	write (fd, “\n”, 1);
	fsync (fd);
	close (fd);
}
```

`fdatasync`也可以实现类似功能。不像前者，`fdatasync`无法保证更新文件的“修改时间”,但是运行速度更快。

利用`fsync`可以实现同步I/O，保证所有写入按顺序立刻执行。这一文件必须用`open`打开，并传入`O_SYNC`选项。

# 5 `getrlimit`和`setrlimit`——限制资源 

Linux里有一个`ulimit`命令可以限制程序对系统资源的获取。Linux还提供了`getrlimit`和`setrlimit`两个系统调用可以实现类似的功能。

每类资源有两种限制——硬性限制和软性限制。软性限制不会超过硬性限制。只有Root用户可以改变硬性限制。

`getrlimit`和`setrlimit`调用的第一个参数都是要限制的资源类型；第二个参数是一个`rlimit`结构体变量指针。`rlimit`结构体有两个域：`rlim_cur`是软性限制，`rlim_max`是硬性限制。

对于第一个参数，最常限制的资源有：

* `RLIMIT_CPU`，程序使用的最大CPU时间，单位为秒。程序如果运行超时，就会被`SIGXCPU`信号中止；
* `RLIMIT_DATA`，程序的最大内存占用。超出限制的申请都会被系统否决。
* `RLIMIT_NPROC`，程序的最多子进程数量。超出限制的fork调用都会失败。
* `RLIMIT_NOFILE`，程序的最多同时打开的文件描述符数量。
* 更多内容可以查看`setrlimit`的man手册页。

示例，设置CPU限制。

```C
#include <sys/resource.h> 
#include <sys/time.h> 
#include <unistd.h>

int main () 
{
	struct rlimit rl;
	
	/* Obtain the current limits. */ 
	getrlimit (RLIMIT_CPU, &rl);
	
	/* Set a CPU limit of 1 second. */ 
	rl.rlim_cur = 1;
	setrlimit (RLIMIT_CPU, &rl); 
	
	/* Do busy work. */
	while (1);
	return 0;
}

```

# 6 `getrusage`——进程统计

`getrusage`可以从内核获取进程的运行情况统计。第一个参数：如果是`RUSAGE_SELF`，就只统计进程本身；如果是`RUSAGE_CHILDREN`，就会统计进程所衍生出的所有子进程。第二个参数是一个`rusage`结构体变量指针，用来放置结果。

`rusage`结构体含有三个域：

* `ru_utime`——一个`timeval`结构体，表示进程执行用户程序的时间，单位为秒。
* `ru_stime`——一个`timeval`结构体，表示进程执行系统调用的时间，单位为秒。
* `ru_maxrss`——进程在执行过程中，一度拥有的最大物理内存。

```C
#include <stdio.h> 
#include <sys/resource.h> 
#include <sys/time.h> 
#include <unistd.h>

void print_cpu_time() 
{
	struct rusage usage;
	getrusage (RUSAGE_SELF, &usage);
	printf (“CPU time: %ld.%06ld sec user, %ld.%06ld sec system\n”,
				usage.ru_utime.tv_sec, usage.ru_utime.tv_usec, 
				usage.ru_stime.tv_sec, 	usage.ru_stime.tv_usec);
}
```

# 7 `gettimeofday`——系统时钟

`gettimeofday`调用可以直接读取系统时钟。调用时需要传入一个`timeval`类型结构体变量指针。这一结构体可以保存系统时钟，单位为秒。`timeval`类型结构体有两个域：`tv_sec`域里面是秒数，`tv_usec`域里面是毫秒数。`timeval`类型结构体里的数字是指从1970年1月1日UTC 0:00起算的累加时间。该调用在`<sys/time.h>`中声明。

`timeval`类型结构体中的数据并不直观。可以使用`localtime`和`strftime`库函数来做进一步处理。`localtime`可以把`timeval`结构体的`tv_sec`域处理为`tm`结构体。而`strftime`可以把`tm`结构体中的数据输出为格式化的字符串。

```C
#include <stdio.h> 
#include <sys/time.h> 
#include <time.h> 
#include <unistd.h>

void print_time () 
{
	struct timeval tv; 
	struct tm* ptm;
	char time_string[40]; 
	long milliseconds;
	
	/* Obtain the time of day, and convert it to a tm struct. */ 
	gettimeofday (&tv, NULL);
	ptm = localtime (&tv.tv_sec);
	
	/* Format the date and time, down to a single second. */
	strftime (time_string, sizeof (time_string), “%Y-%m-%d %H:%M:%S”, ptm); 
	
	/* Compute milliseconds from microseconds. */
	milliseconds = tv.tv_usec / 1000;
	
	/* Print the formatted time, in seconds, followed by a decimal point and the milliseconds. */
	printf (“%s.%03ld\n”, time_string, milliseconds);
}
```

# 8 `mlock`系列调用——锁定物理内存

`mlock`系列调用可以使程序锁定其部分或全部地址空间所对应的物理内存。这样，即使程序有一段时间没有访问这些页面，系统也不会把他们调换出去。

这一功能对于一些难以预测内存使用的程序很有用。另外，还有一些对安全有很高要求的程序，需要避免某些数据在系统调换页面时被写入磁盘。因为这些写入磁盘的数据有可能在程序结束后被别有用心的从磁盘文件中恢复。

锁定内存很简单，向`mlock`传入内存起始地址和内存总长即可。下例展示了如何锁定32MB内存空间。

```C
const int alloc_size = 32 * 1024 * 1024; 
char* memory = malloc (alloc_size); 
mlock (memory, alloc_size);
```

上述操作并不保证系统为调用进程单独保留页面，因为系统会执行“写时复制”策略。为了保证调用进程能有独享的内存空间，可以执行下列代码：

```C
size_t i;
size_t page_size = getpagesize ();
for (i = 0; i < alloc_size; i += page_size)
	memory[i] = 0;
```

调用`munlock`来解锁物理内存。

调用`mlockall`一次性锁定太多内存会非常危险。因为，其他进程就只能有少量内存资源。为了保证正常运行，系统不得不频繁的调换页面。严重时，会导致内存抖动(thrashing)，从而大幅降低系统性能。随意最好渐进地锁定物理内存。

因此，调用`mlock`、`mlockall`的权限被限制在root用户手里。非root进程试图调用`mlock`族函数时，会出错，调用返回-1，并将ERRNO设置为EPERM。

`munlockall`可以把所有被锁定的物理内存都释放掉。

使用`top`命令可以查看进程的内存使用情况。`SIZE`栏里是进程的虚拟内存页面大小；`RSS`栏里是进程驻留在物理内存的大小。

`mlock`系列调用声明在`<sys/mman.h>`文件中。

# 9 `mprotect`——设置内存权限

前面介绍了`mmap`调用，可以将文件映射到内存中。`mmap`调用第三个参数可以将页面权限设置为`PROT_READ`、`PROT_WRITE`、`PROT_EXEC`、`PROT_NONE`，分别对应于“读”，“写”，“执行”以及“禁止”。

当内存被声明之后，可以使用`mprotect`调用来修改内存权限，传入参数：内存地址、内存长度以及一组保护位参数。需要注意的是，内存地址必须与页面大小对齐，内存长度必须是页面大小的整数倍。

>**获取对齐的页面**
>
>通常，`malloc`获取的内存地址是没有对齐的，即使长度是页面大小的整数倍。所以要使用`mprotect`来修改内存权限，必须声明一个超过需求的内存区域。
>

使用方法演示。

```C
# Map file to memory
int fd = open (“/dev/zero”, O_RDONLY);
char* memory = mmap (NULL, page_size, PROT_READ | PROT_WRITE,
						MAP_PRIVATE, fd, 0);
close (fd);

# change the access method
mprotect (memory, page_size, PROT_READ);
```

完整示例。

```C
#include <fcntl.h> 
#include <signal.h> 
#include <stdio.h> 
#include <string.h> 
#include <sys/mman.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

static int alloc_size; 
static char* memory;
void segv_handler (int signal_number) 
{
	printf (“memory accessed!\n”);
	mprotect (memory, alloc_size, PROT_READ | PROT_WRITE); 
}

int main () 
{
	int fd;
	struct sigaction sa;
	
	/* Install segv_handler as the handler for SIGSEGV. */ 
	memset (&sa, 0, sizeof (sa));
	sa.sa_handler = &segv_handler;
	sigaction (SIGSEGV, &sa, NULL);

	/* Allocate one page of memory by mapping /dev/zero. Map the memory as write-only, initially. */
	alloc_size = getpagesize ();
	fd = open (“/dev/zero”, O_RDONLY);
	memory = mmap (NULL, alloc_size, PROT_WRITE, MAP_PRIVATE, fd, 0); 
	close (fd);

	/* Write to the page to obtain a private copy. */
	memory[0] = 0;

	/* Make the memory unwritable. */
	mprotect (memory, alloc_size, PROT_NONE);
	/* Write to the allocated memory region. */ 
	memory[0] = 1;
	/* All done; unmap the memory. */ 
	printf (“all done\n”);
	munmap (memory, alloc_size); 
	
	return 0;
}
```

# 10 `nanosleep`——高精度休眠调用

`nanosleep`是UNIX系统的休眠指令`sleep`调用的高精度版本。

`nanosleep`的参数是一个`timespec`结构体指针。其计时精度能够达到10毫秒级别。这一特性能够帮助程序调度平凡执行的短时任务。`timespec`结构体有两个域：`tv_sec`里面是整数秒时，`tv_nsec`是其余的毫秒数。`tv_nsec`的数值不能超过10<sup>9</sup>。

`nanosleep`的另一个特点是容易恢复。如果系统在运行`sleep`或`nanosleep`时，收到信号中断，调用会立刻返回-1，并将ERRON设置为EINTR。如果是`nanosleep`的第二个参数——一个`timespec`结构体指针非NULL，那么`nanosleep`调用会在这个结构体里存储剩余时间。当`nanosleep`返回后，系统可以借助这个结构体做进一步判断。

```C
#include <errno.h> 
#include <time.h>

int better_sleep (double sleep_time) 
{
	struct timespec tv;
	/* Construct the timespec from the number of whole seconds... */ 
	tv.tv_sec = (time_t) sleep_time;
	/* ... and the remainder in nanoseconds. */
	tv.tv_nsec = (long) ((sleep_time - tv.tv_sec) * 1e+9);
	while (1) {
		/* Sleep for the time specified in tv. 
		 * If interrupted by a signal, 
		 * place the remaining time left to sleep back into tv. 
		 */
	int rval = nanosleep (&tv, &tv); 
	if (rval == 0)
		/* Completed the entire sleep time; all done. */
		return 0;
	else if (errno == EINTR)
		/* Interrupted by a signal. Try again. */
		continue; 
	else
		/* Some other error; bail out. */
		return rval; 
	}
	
	return 0;
}
```

# 11 `readlink`——读取符号链接

`readlink`能够读取符号链接所指向的本源。需要传入三个参数：符号链接的路径，用于追溯本源的缓存以及缓存长度。

>Unusually, readlink does not NUL-terminate the target path that it fills into the buffer. It does, however, return the number of characters in the target path, so NUL-terminating the string is simple.

如果第一个参数并非符号链接，调用就会返回-1，并将ERRNO设置为`EINVL`。

```C
#include <errno.h> 
#include <stdio.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	char target_path[256]; 
	char* link_path = argv[1];
	
	/* Attempt to read the target of the symbolic link. */
	int len = readlink (link_path, target_path, sizeof (target_path));
	if (len == -1) {
		/* The call failed. */ 
		if (errno == EINVAL)
			/* It’s not a symbolic link; report that. */
			fprintf (stderr, “%s is not a symbolic link\n”, link_path);
		else
			/* Some other problem occurred; print the generic message. */
			perror (“readlink”); 
		return 1;
	
	} else {
		/* NUL-terminate the target path. */ 
		target_path[len] = ‘\0’;
		
		/* Print it. */
		printf (“%s\n”, target_path);
		
		return 0; 
	}
}

```

# 12 `sendfile`——文件快速发送器

`sendfile`可以将一个文件描述符里的内容拷贝到另一个文件描述符里。这个描述符对应的可以是磁盘文件，也可以是套接字或者设备文件。

通常的文件拷贝方式是先将文件内容拷贝到内存的缓存区中，然后从缓存区拷贝到磁盘中。这个方法效率偏低，中途需要额外的内存区域，还需要两次拷贝过程。

使用`sendfile`方法可以省去中间的缓存要求。需要传入，读写文件两个描述符，一个读取偏离量，一个需要传送的字节长度。读取偏离量是指读取文件从文件开始(0)算起应该开始读取的位置。

`sendfile`调用声明在`<sys/sendfile.h>`中。

```C
#include <fcntl.h> 
#include <stdlib.h> 
#include <stdio.h> 
#include <sys/sendfile.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	int read_fd;
	int write_fd;
	struct stat stat_buf; 
	off_t offset = 0;

	/* Open the input file. */
	read_fd = open (argv[1], O_RDONLY);
	
	/* Stat the input file to obtain its size. */
	fstat (read_fd, &stat_buf);
	
	/* Open the output file for writing, 
	 * with the same permissions as the source file. */
	write_fd = open (argv[2], O_WRONLY | O_CREAT, stat_buf.st_mode); 
	
	/* Blast the bytes from one file to the other. */
	sendfile (write_fd, read_fd, &offset, stat_buf.st_size);

	/* Close up. */
	close (read_fd);
	close (write_fd);

	return 0;
}
```

# 13 `setitimer`——设置间隔时间

`setitimer`可以帮助进程在未来一个固定时间后发送一个信号。

有三种计时器：

* `ITIMER_REAL`，进程在固定时间后发送一个`SIGALARM`信号。这个时间段根据系统时钟计算。
* `ITIMER_VIRTUAL`，进程在固定时间后发送一个`SIGVTALARM`信号。如果在这之间，进程暂停的时间(哪怕是进程陷入内核的时间也不算)不计入其中。
* `ITIMER_PROF`，进程在固定时间后发送一个`SIGPROF`信号。进程运行的时间或者进程陷入内核的时间都计入其中。

`setitimer`的第一个参数就是上述计时器类型；第二个参数是一个`itimerval`结构体指针；第三个参数如果不是NULL，那可以放入另一个计时器的`itimerval`结构体指针。

`itimerval`结构体有两个域：`it_value`是一个`timeval`结构体，里面包含了计时器失效时间，如果为0，则计时器无效；`it_interval`也是一个`timeval`结构体，决定了计时器超时之后的行为：如果为0；计时器超时之后立刻失效，如果为一个非零正数，计时器超时之后会以新的时间循环计时。

```C
#include <signal.h> 
#include <stdio.h> 
#include <string.h> 
#include <sys/time.h>

void timer_handler (int signum) 
{
	static int count = 0;
	printf (“timer expired %d times\n”, ++count); 
}

int main () 
{
	struct sigaction sa; 
	struct itimerval timer;
	
	/* Install timer_handler as the signal handler for SIGVTALRM. */
	memset (&sa, 0, sizeof (sa));
	sa.sa_handler = &timer_handler;
	sigaction (SIGVTALRM, &sa, NULL);

	/* Configure the timer to expire after 250 msec... */ 
	timer.it_value.tv_sec = 0;
	timer.it_value.tv_usec = 250000;
	
	/* ... and every 250 msec after that. */ 
	timer.it_interval.tv_sec = 0; 
	timer.it_interval.tv_usec = 250000;

	/* Start a virtual timer. It counts down whenever this process is executing. */
	setitimer (ITIMER_VIRTUAL, &timer, NULL);
	
	/* Do busy work. */
	while (1);
	
	/* 
	 * NO RETURN ?? 
	 */
}

```

# 14 `sysinfo`——获取系统统计信息

`sysinfo`能够将系统信息写入`sysinfo`结构体。该结构体有四个域：

* `uptime`——自系统启动之后过去的时间，单位为秒
* `totalram`——可用物理内存的总计
* `freeram`——没有使用的物理内存
* `procs`——系统进程数量

`sysinfo`结构体详情可查Linux手册`sysinfo`页。

要使用`sysinfo`调用，必须引入`<linux/kernel.h>`，`<linux/sys.h>`和`<sys/sysinfo.h>`三个文件。

```C
#include <linux/kernel.h> 
#include <linux/sys.h> 
#include <stdio.h> 
#include <sys/sysinfo.h>

int main ()
{
	/* Conversion constants. */
	const long minute = 60;
	const long hour = minute * 60;
	const long day = hour * 24;
	const double megabyte = 1024 * 1024; 
	
	/* Obtain system statistics. */ 
	struct sysinfo si;
	sysinfo (&si);

	/* Summarize interesting values. */
	printf (“system uptime : %ld days, %ld:%02ld:%02ld\n”,
				si.uptime / day, (si.uptime % day) / hour,
				(si.uptime % hour) / minute, si.uptime % minute); 
	
	printf (“total RAM : %5.1f MB\n”, si.totalram / megabyte); 
	printf (“free RAM : %5.1f MB\n”, si.freeram / megabyte); 
	printf (“process count : %d\n”, si.procs);

	return 0;
}
```

# 15 `uname`

`uname`调用可以将系统信息填入`utsname`结构体中。`uname`调用定义在`<sys/utsname.h>`中。

`utsname`结构体中有六个域：

* `sysname`——系统名称，比如Linux
* `release` `version`——系统的发行号和版本级别
* `machine`——硬件平台信息，对于x86 Linux，这里就是i386或i686
* `node`——系统主机名
* `__domain`——系统域名

```C
#include <stdio.h> 
#include <sys/utsname.h>

int main () 
{
	struct utsname u;
	uname (&u);
	printf (“%s release %s (version %s) on %s\n”, u.sysname, u.release,
													u.version, u.machine); 
	
	return 0;
}

```



























