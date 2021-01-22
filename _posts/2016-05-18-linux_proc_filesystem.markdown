---
layout: post
title: Advanced Linux Programming - proc文件系统
date: 2016-05-18 19:53:02 +8000
categories: 翻译
---
 

在shell终端里不带任何参数，直接运行`mount`命令可以显示正在挂载的文件系统。其中有这么一行

```shell
none on /proc type proc (rw)
```

这就是`/proc`文件系统。第一个域显示`none`，说明这个文件没有和任何硬件设备挂钩。`/proc`文件系统实际上是一个通向Linux内核的窗口，看起来像一个能够向内核提供参数、数据结构、统计信息等的文件。`/proc`文件系统的内容是随内核运行变化的。用户进程还可以通过改变`/proc`文件系统内容来改变内核的设置。

在Linux手册的`proc(5)`项里查看`/proc`文件系统的详细介绍。其源码位于`/usr/src/linux/fs/proc/`。

# 1 获取信息

`/proc`文件系统中的信息大多都是可读的，也很容易解析。例如，`/proc/cpuinfo`含有系统CPU的信息。

```shell
$ cat /proc/cpuinfo

processor	    : 0
vendor_id	    : GenuineIntel
cpu family    	: 6
model	    	: 61
model name    	: Intel(R) Core(TM) i5-5250U CPU @ 1.60GHz
stepping	    : 4
cpu MHz	    	: 1599.999
cache size    	: 3072 KB
physical id    	: 0
siblings	    : 1
core id	    	: 0
cpu cores    	: 1
apicid		    : 0
initial apicid	: 0
fpu		        : yes
fpu_exception	: yes
cpuid level    	: 20
wp	        	: yes
flags	    	: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 syscall nx rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc pni pclmulqdq monitor ssse3 cx16 sse4_1 sse4_2 movbe popcnt aes xsave avx rdrand hypervisor lahf_lm abm 3dnowprefetch rdseed
bugs	    	:
bogomips    	: 3199.99
clflush size	: 64
cache_alignment	: 64
address sizes	: 39 bits physical, 48 bits virtual
power management:
```

在C语言中提取文件信息的一个简单办法是：先将文件读取到缓存中，然后调用`sscanf`函数在内存中解析数据。

```C
#include <stdio.h>
#include <string.h>

/* Returns the clock speed of the system’s CPU in MHz, as reported by /proc/cpuinfo. 
 * On a multiprocessor machine, returns the speed of the first CPU. 
 * On error returns zero. 
 */
float get_cpu_clock_speed () 
{
	FILE* fp;
	char buffer[1024]; 
	size_t bytes_read; 
	char* match;
	float clock_speed;
	
	/* Read the entire contents of /proc/cpuinfo into the buffer. */ 
	fp = fopen (“/proc/cpuinfo”, “r”);
	bytes_read = fread (buffer, 1, sizeof (buffer), fp);
	fclose (fp);
	
	/* Bail if read failed or if buffer isn’t big enough. */ 
	if (bytes_read == 0 || bytes_read == sizeof (buffer))
		return 0;
	
	/* NUL-terminate the text. */
	buffer[bytes_read] = ‘\0’;
	
	/* Locate the line that starts with “cpu MHz”. */ 
	match = strstr (buffer, “cpu MHz”);
	if (match == NULL)
		return 0;
		
	/* Parse the line to extract the clock speed. */ 
	sscanf (match, “cpu MHz : %f”, &clock_speed); 
	
	return clock_speed;
}

int main () 
{
	printf (“CPU clock speed: %4.0f MHz\n”, get_cpu_clock_speed ());
	return 0;
}
```

不同版本Linux的`/proc`文件系统不尽相同。编写跨版本程序时一定要考虑到这个问题。

# 2 进程条目

每一个正在运行的进程都在`/proc`文件系统对应一个文件夹。文件夹名称就是相应进程的ID。这些文件夹随着进程的生成而建立，并随进程的终止而移除。这些文件夹里的文件都描述着对应进程的信息：

* `cmdline`里是进程的参数表。
* `cwd`是一个指向当前进程工作目录的链接符号。
* `environ`包含了进程的运行环境信息。
* `exe`是一个指向进程正在执行的可执行文件的链接符号。
* `fd`是一个子目录，里面都是进程所打开的文件描述符。 
* `maps`里都是映射到当前进程的地址空间的文件的信息，例如文件名称、所映射的内存地址以及这些地址的访问权限。
* `root`是指向进程根目录的链接符号. 进程根目录通常就是系统根目录`/`。进程在进程根目录可以调用超级用户(chroot)命令或系统调用。
* `stat`包含有进程的运行状态信息和统计数据。这些内容很难读懂，但容易用代码解析。 
* `statm`包含有进程的内存使用信息。
* `status`包含有进程的运行状态信息和统计数据，而且方便调阅。
* 只有SMP Linux系统会有一个`cpu`条目，里面是“breakdown of process time (user and system) by CPU“。

出于安全考虑，其中一些条目只有进程所有者或超级用户具有访问权限。

## `/proc/self`

每条进程调用`/proc/self`都会指向各自的`/proc/<pid>`目录。

下面代码展示了如何利用`/proc`文件系统实现`getpid`函数，即获取调用进程的PID。

```C
#include <stdio.h> 
#include <sys/types.h> 
#include <unistd.h>

/* Returns the process ID of the calling processes, as determined from the /proc/self symlink. */
pid_t get_pid_from_proc_self () 
{
	char target[32];
	int pid;
	
	/* Read the target of the symbolic link. */ 
	readlink (“/proc/self”, target, sizeof (target));
	
	/* The target is a directory named for the process ID. */ 
	sscanf (target, “%d”, &pid);
	
	return (pid_t) pid;
}

int main () 
{
	printf (“/proc/self reports process id %d\n”, (int) get_pid_from_proc_self ());
	printf (“getpid() reports process id %d\n”, (int) getpid ());
	return 0;
}
```

## 进程参数表

每一个进程的`/proc/<pid>/cmdline`里有进程的参数表。里面的参数都是单个字符，并用`NUL`隔开。不过问题是，大部分字符串函数无法处理字符串中间夹杂的`NUL`。

>**`NUL` vs `NULL`**
>`NUL`是一个字符，整数值为0。而`NULL`是一个值为0的指针。
>
>C语言中，所有字符串的结尾都是`NUL`。所以字符串`Hello, world!`虽然只有13个字符，但在C语言眼里中，这里有14个字符。因为还有一个`NUL`在感叹号之后。在C语言中，`NUL`被定义为`\0`或`(char) 0`。
>
>`NULL`是一个不指向内存区域的指针，也就是空指针。在C语言中，`NULL`被定义为`((void*)0)`。

下面程序能够读取指定PID的进程的参数表。

```C
#include <fcntl.h> 
#include <stdio.h> 
#include <stdlib.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

/* Prints the argument list, one argument to a line, of the process given by PID. */
void print_process_arg_list (pid_t pid) 
{
	int fd;
	char filename[24]; 
	char arg_list[1024]; 
	size_t length;
	char* next_arg;
	
	/* Generate the name of the cmdline file for the process. */
	snprintf (filename, sizeof (filename), “/proc/%d/cmdline”, (int) pid); 
	
	/* Read the contents of the file. */
	fd = open (filename, O_RDONLY);
	length = read (fd, arg_list, sizeof (arg_list));
	close (fd);
	
	/* read does not NUL-terminate the buffer, so do it here. */ 
	arg_list[length] = ‘\0’;
	
	/* Loop over arguments. Arguments are separated by NULs. */ 
	next_arg = arg_list;
	while (next_arg < arg_list + length) {
		/* Print the argument. Each is NUL-terminated, 
		 * so just treat it like an ordinary string. 
		 */
		printf (“%s\n”, next_arg);
		
		/* Advance to the next argument. Since each argument is NUL-terminated, 
		 * strlen counts the length of the next argument, not the entire argument list. 
		 */ 
		 next_arg += strlen (next_arg) + 1;
	} 
}

int main (int argc, char* argv[]) 
{
	pid_t pid = (pid_t) atoi (argv[1]); 
	print_process_arg_list (pid); 
	
	return 0;
}
```

使用：

```shell
$ ps 372

PID TTY STAT TIME COMMAND
372 ? S 0:00 syslogd -m 0

$ ./print-arg-list 372 

syslogd
-m
0
```

## 进程环境

`environ`里包含了进程的运行环境信息。和`cmdline`一样，`environ`里的环境信息由`NUL`分隔。

```C
#include <fcntl.h> 
#include <stdio.h> 
#include <stdlib.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

/* Prints the environment, one environment variable to a line, of the process given by PID. */
void print_process_environment (pid_t pid) 
{
	int fd;
	char filename[24];
	char environment[8192]; 
	size_t length;
	char* next_var;
	
	/* Generate the name of the environ file for the process. */
	snprintf (filename, sizeof (filename), “/proc/%d/environ”, (int) pid); 
	
	/* Read the contents of the file. */
	fd = open (filename, O_RDONLY);
	length = read (fd, environment, sizeof (environment));
	close (fd);

	/* read does not NUL-terminate the buffer, so do it here. */ 
	environment[length] = ‘\0’;
	
	/* Loop over variables. Variables are separated by NULs. */ 
	next_var = environment;

	while (next_var < environment + length) {
		/* Print the variable. Each is NUL-terminated, so just treat it like an ordinary string. */
		printf (“%s\n”, next_var);
		
		/* Advance to the next variable. Since each variable is
		 * NUL-terminated, strlen counts the length of the next variable,
		 * not the entire variable list. 
		 */ 
		 next_var += strlen (next_var) + 1;
	} 
}

int main (int argc, char* argv[]) 
{
	pid_t pid = (pid_t) atoi (argv[1]); 
	print_process_environment (pid); 
	return 0;
}
```

上面代码展示了如何读取并解析`environ`里的信息。

## 进程可执行文件

尽管，一般来讲shell命令的第一个参数就是进程的执行文件，但是有时其参数可以可能改变实际运行的执行文件。因此，`/proc/<pid>/exe`中的信息更为可靠。

下面代码可以获取进程的执行文件目录。

```C
#include <limits.h> 
#include <stdio.h> 
#include <string.h> 
#include <unistd.h>

/* Finds the path containing the currently running program executable. 
 * The path is placed into BUFFER, which is of length LEN. 
 * Returns the number of characters in the path, or -1 on error. 
 */
size_t get_executable_path (char* buffer, size_t len) 
{
	char* path_end;
	
	/* Read the target of /proc/self/exe. */
	if (readlink (“/proc/self/exe”, buffer, len) <= 0)
		return -1;
	
	/* Find the last occurrence of a forward slash, the path separator. */ 
	path_end = strrchr (buffer, ‘/’);
	if (path_end == NULL)
		return -1;
	
	/* Advance to the character past the last slash. */
	++path_end;

	/* Obtain the directory containing the program 
	 * by truncating the path after the last slash. 
	 */
	*path_end = ‘\0’;

	/* The length of the path is the number of characters up through the last slash. */
	return (size_t) (path_end - buffer);
}

int main () 
{
	char path[PATH_MAX];
	get_executable_path (path, sizeof (path));
	printf (“this program is in the directory %s\n”, path); 
	
	return 0;
}
```

## 进程文件描述符

`fd`子目录里包含了指向进程所打开文件的描述符的链接符号。对这些链接符号执行读写操作也就相当于对相应文件的读写操作。目录里每个子项的名称都是文件描述符打开的序号。

```shell
$ vim &       

[1] 16

$ ps axw

  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 /bin/bash
   16 ?        T      0:00 vim
   19 ?        R+     0:00 ps axw

[1]+  Stopped                 vim

$ ls -l /proc/16/fd
total 0
lrwx------ 1 root root 64 May 18 05:49 0 -> /0
lrwx------ 1 root root 64 May 18 05:49 1 -> /0
lrwx------ 1 root root 64 May 18 05:48 2 -> /0
```

根据之前的内容可以知道，进程的文件描述符`0``1``2`分别对应于进程自己的stdin、stdout、stderr。

所以向`/proc/16/fd/1`写入的文本，实际上可以在屏幕中显示出来。

```shell
$ echo “Hello, world.” >> /proc/16/fd/1
```

如果进程继续打开文件，对应的文件描述符也会出现在`fd`目录中。

```C
#include <fcntl.h> 
#include <stdio.h> 
#include <sys/stat.h> 
#include <sys/types.h> 
#include <unistd.h>

int main (int argc, char* argv[]) 
{
	const char* const filename = argv[1];
	int fd = open (filename, O_RDONLY);
	printf (“in process %d, file descriptor %d is open to %s\n”,
				(int) getpid (), (int) fd, filename); 
	while (1);
	
	return 0;
}
```

将上述代码编译为open-and-spin。在终端中执行程序。

```shell
$ ./open-and-spin /etc/fstab

in process 2570, file descriptor 3 is open to /etc/fstab
```

在另一终端里查看`fd`目录。

```shell
% ls -l /proc/2570/fd

total 0
lrwx------ 1 samuel samuel 64 Jan 30 01:30 0 -> /dev/pts/2
lrwx------ 1 samuel samuel 64 Jan 30 01:30 1 -> /dev/pts/2
lrwx------ 1 samuel samuel 64 Jan 30 01:30 2 -> /dev/pts/2
lr-x------ 1 samuel samuel 64 Jan 30 01:30 3 -> /etc/fstab
```

注意最后一项，最新打开的文件描述符名称为3。

## 进程内存统计

`statm`里面含有7个数字，分别用空格分开。每一项都是进程某些内容占用内存的情况统计。

* 总进程大小
* 进程驻留在物理内存的大小
* 于其他进程分享的内存大小
* 进程文本大小——也就是可执行文件的代码大小
* 映射到进程的共享库大小
* 进程栈所使用的内存大小
* 脏页面大小——也就是被进程修改过的页面大小

## 进程统计

`status`项里是一些具有可读性的进程情况统计。主要有进程ID，父进程ID，真实或有效的用户ID、组ID，内存使用以及决定是否捕捉、忽略或阻塞某些信号的位掩码。


# 3 硬件信息

`/proc/cpuinfo`可以查看CPU信息。

`/proc/devices`可以查看设备信息。

`/proc/pci`可以查看PCI总线信息。

`/proc/tty/driver/serial`可以查看系统串口信息。

# 4 内核信息

`/proc/version`查看内核版本信息。

```shell
$ cat /proc/version

Linux version 4.1.13-boot2docker (root@11aafb97cfeb) (gcc version 4.9.2 (Debian4.9.2-10) ) #1 SMP Fri Nov 20 19:05:50 UTC 2015

$ cat /proc/sys/kernel/ostype

Linux

$ cat /proc/sys/kernel/osrelease

4.1.13-boot2docker

$ cat /proc/sys/kernel/version

#1 SMP Fri Nov 20 19:05:50 UTC 2015
```

`/proc/sys/kernel/hostname`和`/proc/sys/kernel/domainname`分别记录的系统的主机名和域名。

`/proc/meminfo`理由系统的内存使用情况统计。

```shell

$ cat /proc/meminfo

MemTotal:        2050728 kB
MemFree:         1825280 kB
MemAvailable:    1849244 kB
Buffers:           18684 kB
Cached:           139328 kB
SwapCached:            0 kB
Active:            85608 kB
Inactive:          97132 kB
Active(anon):      58724 kB
Inactive(anon):    92996 kB
Active(file):      26884 kB
Inactive(file):     4136 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       1448436 kB
SwapFree:        1448436 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:         24768 kB
Mapped:            30524 kB
Shmem:            126996 kB
Slab:              26008 kB
SReclaimable:      13964 kB
SUnreclaim:        12044 kB
KernelStack:        2080 kB
PageTables:          836 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     2473800 kB
Committed_AS:     244360 kB
VmallocTotal:   34359738367 kB
VmallocUsed:        9960 kB
VmallocChunk:   34359698060 kB
AnonHugePages:     14336 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       49088 kB
DirectMap2M:     2048000 kB
```

# 5 文件系统

`/proc/filesystems`里是当前挂载到Linux内核的文件系统列表。如果是动态地挂载或卸载的文件系统，不一定能够在这里看到。

`/proc/mounts`是当前挂载到Linux内核的文件系统的详情列表。包含有挂载描述符、挂载设备、挂载点以及其他信息。

```shell
$ cat /proc/filesystems 

nodev	sysfs
nodev	rootfs
nodev	ramfs
nodev	bdev
nodev	proc
nodev	cgroup
nodev	cpuset
nodev	tmpfs
nodev	devtmpfs
nodev	debugfs
nodev	securityfs
nodev	sockfs
nodev	pipefs
nodev	devpts
	ext3
	ext2
	ext4
nodev	hugetlbfs
	vfat
nodev	ecryptfs
	fuseblk
nodev	fuse
nodev	fusectl
nodev	pstore
nodev	mqueue
nodev	binfmt_misc
nodev	mtd_inodefs

$ cat /proc/mounts 

sysfs /sys sysfs rw,nosuid,nodev,noexec,relatime 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
udev /dev devtmpfs rw,relatime,size=621336k,nr_inodes=155334,mode=755 0 0
devpts /dev/pts devpts rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000 0 0
tmpfs /run tmpfs rw,nosuid,noexec,relatime,size=127248k,mode=755 0 0
/dev/disk/by-uuid/8ce1d249-5033-42c5-bb7f-da454b447e60 / ext4 rw,relatime,errors=remount-ro,data=ordered 0 0
none /sys/fs/cgroup tmpfs rw,relatime,size=4k,mode=755 0 0
none /sys/fs/fuse/connections fusectl rw,relatime 0 0
none /sys/kernel/debug debugfs rw,relatime 0 0
none /sys/kernel/security securityfs rw,relatime 0 0
none /run/lock tmpfs rw,nosuid,nodev,noexec,relatime,size=5120k 0 0
none /run/shm tmpfs rw,nosuid,nodev,relatime 0 0
none /run/user tmpfs rw,nosuid,nodev,noexec,relatime,size=102400k,mode=755 0 0
none /sys/fs/pstore pstore rw,relatime 0 0
binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc rw,nosuid,nodev,noexec,relatime 0 0
systemd /sys/fs/cgroup/systemd cgroup rw,nosuid,nodev,noexec,relatime,name=systemd 0 0
gvfsd-fuse /run/user/1000/gvfs fuse.gvfsd-fuse rw,nosuid,nodev,relatime,user_id=1000,group_id=1000 0 0
```

`/proc/locks`显示了系统所知的文件锁。文件所的使用可见下章。(P164)

# 6 系统统计

`/proc/loadavg`展示了系统的运行信息。其前三个数字分别是系统在过去1分钟、5分钟、15分钟内活跃的运行进程；第四个数字是系统当前可调度进程，而没有被阻塞的进程；第五个数字是系统的运行进程总计；第六个是最后一次运行进程ID。

`/proc/uptime`所显示的分别是系统启动后的运行时间，系统闲置时间。`uptime`命令和`sysinfo`函数也同样可以查看系统的时间信息。

```C
#include <stdio.h>

/* Summarize a duration of time to standard output. 
 * TIME is the amount of time, in seconds, and LABEL is a short descriptive label. 
 */
void print_time (char* label, long time) 
{
	/* Conversion constants. */ 
	const long minute = 60;
	const long hour = minute * 60; 
	const long day = hour * 24;
	
	/* Produce output. */
	printf (“%s: %ld days, %ld:%02ld:%02ld\n”, label, time / day,
			(time % day) / hour, (time % hour) / minute, time % minute);
}

int main () 
{
	FILE* fp;
	double uptime, idle_time;
	
	/* Read the system uptime and accumulated idle time from /proc/uptime. */ 
	fp = fopen (“/proc/uptime”, “r”);
	fscanf (fp, “%lf %lf\n”, &uptime, &idle_time);
	fclose (fp);
	
	/* Summarize it. */
	print_time (“uptime “, (long) uptime);
	print_time (“idle time”, (long) idle_time);
	return 0;
}
```


































