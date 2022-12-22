---
layout: post
title: Advanced Linux Programming - 进程间通信
date: 2016-05-15 19:53:02
categories: 翻译
---



第三章“进程”里，父进程向子进程传递消息的方法只有启动子进程的命令行参数和环境变量两条路径；子进程向父进程传递消息只能通过子进程的退出代码。而两个进程同时运行时，也往往有交流的需要。

本章将讨论五种进程间通信(IPC)方法：

1. 内存共享(Shared Memory)允许进程通过向共享的内存区域读写内容来通信；
2. 内存映射(Mapped Memory)和前者类似，只是跟文件系统的某一文件相关联；
3. 管道(Pipes)允许相关联的进程间将前者的结果传递后者做输入参数；
4. FIFO和管道类似允许不相关联的进程间实现通信；
5. 套接字(sockets)支持不同进程之间的通信，即使进程不在同一个电脑上。

# 1 共享内存

共享内存机制允许两条及以上的进程同时读取统一内存区域。这些内存调用`malloc`得到的指针实际上指向同一区域。

## 快速的本地通信手段

共享内存机制的读取速度非常快，和本地内存数据的读取诉的是一致的。

由于Linux内核不会自动同步进程，因此需要做着自行编写同步机制。可以使用信号量机制来实现进程间的同步机制。

## 内存模型

首先需要一条进程声明(allocate)地址空间，然后其余想要通信的进程执行关联(attach)操作。系统内核就会将同一内存段映射在不同进程的虚拟地址空间中。当进程都不再使用这一内存段时，应该声明解除关联(detach)，最后一个进程应该回收共享的内存空间，以防泄漏。

共享的内存空间大小都是页面大小(page size)的倍数，使用`getpagesize`函数可以获取系统的页面大小。

## 声明

进程声明共享内存空间应该使用`shmget`函数。

函数的第一个参数是标记共享内存空间的`int`类型key值。其它想要共享这一内存空间的进程根据这一key值想系统提出申请。但是如果其他进程使用了相同的key就会导致冲突。使用常量`IPC_PRIVATE`可以保证声明的是新的内存空间。

第二个参数指出了共享内存空间的字节数。内存段的最小单位是页面，因此最终分配到的内存空间是这一数值的较大近似值。

第三个参数是`shmget`函数的flag位，标记出函数的特殊选项：

* `IPC_CREAT`表明需要创建新的内存段。
* `IPC_EXCL`，总和`IPC_CREAT`一起使用，如果遇到已经存在的key值，会导致`shmget`函数失败。这一选项总会保证调用进程得到独有的共享内存空间。如果没有这一选项，并使用了已经使用了的key值，函数就会返回已存在的内存段。
* 还有一些选项标记了内存空间的权限分配模式。这些选项值都声明在`<sys/stat.h>`文件中。Linux的man手册`stat`页第二节详细讲述了这些选项。

用法示例：

```C
int segment_id = shmget (shm_key, getpagesize (), 
								IPC_CREAT | S_IRUSR | S_IWUSER);
```

## 关联与解除关联

一个进程程想要关联某个共享内存区域，应该调用`shmat`函数。第一个参数是`shmget`函数返回的共享内存段ID `SHMID`。第二个参数是指定接收映射的内存地址空间。如果是NULL，系统就会自行分配。但三个参数是flag标记：

* `SHM_RND`使得地址空间为约减为页面大小的倍数。如果不使用，则地址空间就会和`shmget`函数第二个参数对齐。
* `SHM_RDONLY`，表明调用进程只读取共享内存段，不写入。

如果调用成功，就会返回关联的内存空间地址。

fork产生的子进程能够继承父进程的共享内存空间，如果不想继续共享，则应该执行解除关联操作。

进程要执行解除关联操作，应该调用`shmdt`函数。向其传入`shmat`函数返回的关联的内存空间地址即可。调用`exit`以及`exec`族都会自动解除关联。

## 控制与回收

`shmctl`函数可以返回共享内存段的信息，并修改之。第一个参数是共享内存的ID。要得到信息，应继续传入`IPC_STAT`标记和`shmid_ds`结构的指针。

要移除内存段，应继续传入`IPC_RMID`标记和NULL指针。然后，内存段会在最后一个进程解除关联后被自动回收。

每一段共享内存都应该被显式地使用`shmctl`函数回收。调用`exit`以及`exec`族都只能解除关联，不能回收内存空间。

## 一个栗子

```C
#include <stdio.h> 
#include <sys/shm.h> 
#include <sys/stat.h>

int main () 
{
	int segment_id;
	char* shared_memory;
	struct shmid_ds shmbuffer;
	int segment_size;
	const int shared_segment_size = 0x6400;

	/* Allocate a shared memory segment. */
	segment_id = shmget (IPC_PRIVATE, shared_segment_size,
							IPC_CREAT | IPC_EXCL | S_IRUSR | S_IWUSR);

	/* Attach the shared memory segment. */
	shared_memory = (char*) shmat (segment_id, 0, 0);
	printf (“shared memory attached at address %p\n”, shared_memory); 
	
	/* Determine the segment’s size. */
	shmctl (segment_id, IPC_STAT, &shmbuffer);
	segment_size = shmbuffer.shm_segsz;
	printf (“segment size: %d\n”, segment_size);

	/* Write a string to the shared memory segment. */
	sprintf (shared_memory, “Hello, world.”);

	/* Detach the shared memory segment. */
	shmdt (shared_memory);
	
	/* Reattach the shared memory segment, at a different address. */ 
	shared_memory = (char*) shmat (segment_id, (void*) 0x5000000, 0); 
	printf (“shared memory reattached at address %p\n”, shared_memory); 
	
	/* Print out the string from shared memory. */
	printf (“%s\n”, shared_memory);
	
	/* Detach the shared memory segment. */ 
	shmdt (shared_memory);
	
	/* Deallocate the shared memory segment. */ 
	shmctl (segment_id, IPC_RMID, 0);

	return 0;
}
```

## 调试

使用`ipcs`命令可以看到进程间通信的相关信息，使用`-m`标记来获得共享内存段的信息。

```shell
$ ipcs

IPC status from <running system> as of <TIME>
T     ID     KEY        MODE       OWNER    GROUP
Message Queues:

T     ID     KEY        MODE       OWNER    GROUP
Shared Memory:
m  65536 0x0052e2c1 --rw-------    <USRNAME>    staff

T     ID     KEY        MODE       OWNER    GROUP
Semaphores:
s  65536 0x0052e2c1 --ra-------    <USRNAME>    staff
s  65537 0x0052e2c2 --ra-------    <USRNAME>    staff
s  65538 0x0052e2c3 --ra-------    <USRNAME>    staff
s  65539 0x0052e2c4 --ra-------    <USRNAME>    staff
s  65540 0x0052e2c5 --ra-------    <USRNAME>    staff
s  65541 0x0052e2c6 --ra-------    <USRNAME>    staff
s  65542 0x0052e2c7 --ra-------    <USRNAME>    staff
s  65543 0x0052e2c8 --ra-------    <USRNAME>    staff

$ ipcs -m

IPC status from <running system> as of <TIME>
T     ID     KEY        MODE       OWNER    GROUP
Shared Memory:
m  65536 0x0052e2c1 --rw-------    <USRNAME>    staff
```

如果共享内存有泄漏，可以使用`ipcrm`命令来回收。

## 小结

共享内存机制是一个快速的双向通信机制，但是必须自己建立同步机制，以避免竞争条件问题。此外，还需要注意做好共享内存ID的管理工作。

# 2 进程信号量

进程信号量机制也称作"System V 信号量机制"。进程信号量的分配、使用和回收与共享内存比较相似。

## 分配与回收

进程可以调用`semget`和`semctl`来分配和回收进程信号量。`semget`函数的第一个参数是信号量集合的key，第二个参数值是集合中的信号量的数量，第三个参数是信号量的权限标记。函数返回的是信号量集合的ID。

进程信号量集合必须显式的回收。向`semctl`传入信号量集合的ID、集合中的信号量的数量、`IPC_RMID`以及`semun`集合类型的任意变量(其实会被忽略)。调用进程的用户ID必须与分配信号量集合进程的用户ID相同。

```C
#include <sys/ipc.h> 
#include <sys/sem.h> 
#include <sys/types.h>

/* We must define union semun ourselves. */
union semun {
	int val;
	struct semid_ds *buf; 
	unsigned short int *array; 
	struct seminfo *__buf;
};

/* Obtain a binary semaphore’s ID, allocating if necessary. */
int binary_semaphore_allocation (key_t key, int sem_flags) 
{
	return semget (key, 1, sem_flags); 
}

/* Deallocate a binary semaphore. 
 * All users must have finished their use. 
 * Returns -1 on failure. 
 */
int binary_semaphore_deallocate (int semid) 
{
	union semun ignored_argument;
	return semctl (semid, 1, IPC_RMID, ignored_argument);
}
```
 
## 初始化

信号量集合的分配和初始化是两回事。进程应该调用`semctl`函数来完成初始化工作。这时，第二个参数应该是0；第三个参数是`SETALL`；第四个参数是一个`semun`集合类型的变量，其`array`域指向一个`unsigned short`类型数组，数组中每个值都将是集合中信号量的初始值。

```C
#include <sys/types.h> 
#include <sys/ipc.h> 
#include <sys/sem.h>

/* We must define union semun ourselves. */
union semun {
	int val;
	struct semid_ds *buf; 
	unsigned short int *array; 
	struct seminfo *__buf;
};

/* Initialize a binary semaphore with a value of 1. */
int binary_semaphore_initialize (int semid) 
{
	union semun argument;
	unsigned short values[1];
	values[0] = 1;
	argument.array = values;
	
	return semctl (semid, 0, SETALL, argument);
}
```

## wait和post操作

进程可以调用`semop`来实施wait和post操作。第一个参数是信号量集合ID；第二个参数是一个`sembuf`结构体数组，指出了想要执行的动作；第三个参数是数组的长度。

`sembuf`结构体有三个域：

* `sem_num`是指要执行动作的信号量在集合中的ID。
* `sem_op`指出了要执行的动作：
	* 如果`sem_op`为正，那么信号量会立即加上这个值。
	* 如果`sem_op`为负，那么信号量会立即减去这个值的绝对值；如果信号量会因此为负，那么进程就会被阻塞，直到其它进程增加信号量，而与之相抵消。
	* 如果`sem_op`为0，那么进程就会阻塞直到信号量为0。
* `sem_flag`是一个flag值。设置为`IPC_NOWAIT`可以防止进程被阻塞。如果已经被阻塞，那么调用`semop`会立即失败。如果设置为`SEM_UNDO`，Linux会在进程退出时自动撤销其曾经的影响。

```C
#include <sys/types.h> 
#include <sys/ipc.h> 
#include <sys/sem.h>

/* Wait on a binary semaphore. 
 * Block until the semaphore value is positive,
 * then decrement it by 1. 
 */
int binary_semaphore_wait (int semid) 
{
	struct sembuf operations[1];
	/* Use the first (and only) semaphore. */ 
	operations[0].sem_num = 0;
	/* Decrement by 1. */ 
	operations[0].sem_op = -1;
	/* Permit undo’ing. */ 
	operations[0].sem_flg = SEM_UNDO;
	
	return semop (semid, operations, 1); 
}

/* Post to a binary semaphore: 
 * increment its value by 1. 
 * This returns immediately. 
 */
int binary_semaphore_post (int semid) 
{
	struct sembuf operations[1];
	/* Use the first (and only) semaphore. */ 
	operations[0].sem_num = 0;
	/* Increment by 1. */ 
	operations[0].sem_op = 1;
	/* Permit undo’ing. */ 
	operations[0].sem_flg = SEM_UNDO;
	
	return semop (semid, operations, 1);
}
```

## 调试

`ipcs -s`命令可以列出所有进程间信号量。调用`ipcrm sem`可以移除信号量集合。

# 3 内存映射

内存映射机制允许进程通过共享文件来实现通信。这一机制类似于，先在内存中声明缓存区，然后将整个文件读入缓存区域。进程可以直接读写缓存。如果缓存发生改动，这些改动还会保存到文件中。

## 映射普通文件

使用`mmap`函数来实现映射。第一个参数是希望用来接受映射的进程内存起始地址，如果用NULL，则有系统自行分配；第二个参数是映射区域的字节数；第三个参数明确了映射内存区域的保护位：`PROT_READ`，` PROT_WRITE`，`PROT_EXEC`分别对应于“读”、“写”、“执行”等许可；第四个参数是一个flag位；第五个参数是所打开文件的描述符；第六个参数是希望开始映射的偏移量，从文件起始开始计算。其中第四个flag位可选项有：

* `MAP_FIXED`：如果使用，Linux就会严格执行所传入的内存起始地址，而不只是作为参考。这个地址必须是页面对齐的。
* `MAP_SHARED`：如果缓存区发生改动，改动会立即直接写入文件。其他进程也可以看到改动。**要使用该机制实现进程通信必须选定该选项**。
* `MAP_PRIVATE`：如果缓存区发生改动，改动不会直接写入文件，而是保存到另一个私有文件中。该选项不能语`MAP_SHARED`合用。

如果调用成功，函数会返回内存区域起始地址的指针。如果调用失败会返回`MAP_FAILED`错误码。

如果不再使用映射，应该调用`munmap`函数释放内存区。调用者应传入内存区域起始地址和区域长度。Linux会在进程退出后自动回收内存。

## 两个栗子

第一个程序`mmap-write`生成随机数，并写入映射文件中。原书P.106-108。

```C
#include <stdlib.h> 
#include <stdio.h> 
#include <fcntl.h> 
#include <sys/mman.h> 
#include <sys/stat.h> 
#include <time.h> 
#include <unistd.h> 

#define FILE_LENGTH 0x100

/* Return a uniformly random number in the range [low,high]. */
int random_range (unsigned const low, unsigned const high) 
{
	unsigned const range = high - low + 1;
	return low + (int) (((double) range) * rand () / (RAND_MAX + 1.0)); 
}

int main (int argc, char* const argv[]) 
{
	int fd;
	void* file_memory;
	
	/* Seed the random number generator. */ 
	srand (time (NULL));
	
	/* Prepare a file large enough to hold an unsigned integer. */ 
	fd = open (argv[1], O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
	lseek (fd, FILE_LENGTH+1, SEEK_SET);
	write (fd, “”, 1);
	lseek (fd, 0, SEEK_SET);
	
	/* Create the memory mapping. */
	file_memory = mmap (0, FILE_LENGTH, PROT_WRITE, MAP_SHARED, fd, 0); 
	close (fd);
	
	/* Write a random integer to memory-mapped area. */ 
	sprintf((char*) file_memory, “%d\n”, random_range (-100, 100));
	
	/* Release the memory (unnecessary because the program exits). */ 
	munmap (file_memory, FILE_LENGTH);
	
	return 0;
}
```
第二个程序`mmap-read`，读取文件中的数字，并打印到屏幕，然后用原数字的双倍来替换。

```C
#include <stdlib.h> 
#include <stdio.h> 
#include <fcntl.h> 
#include <sys/mman.h> 
#include <sys/stat.h> 
#include <unistd.h> 

#define FILE_LENGTH 0x100

int main (int argc, char* const argv[]) 
{
	int fd;
	void* file_memory; 
	int integer;
	
	/* Open the file. */
	fd = open (argv[1], O_RDWR, S_IRUSR | S_IWUSR);

	/* Create the memory mapping. */
	file_memory = mmap (0, FILE_LENGTH, PROT_READ | PROT_WRITE,
							MAP_SHARED, fd, 0);
	close (fd);
	
	/* Read the integer, print it out, and double it. */
	scanf (file_memory, “%d”, &integer);
	printf (“value: %d\n”, integer);
	sprintf ((char*) file_memory, “%d\n”, 2 * integer);

	/* Release the memory (unnecessary because the program exits). */ 
	munmap (file_memory, FILE_LENGTH);
	
	return 0;
}
```

执行程序。

```shell
$ ./mmap-write /tmp/integer-file 
$ cat /tmp/integer-file
42
$ ./mmap-read /tmp/integer-file 
value: 42
$ cat /tmp/integer-file 
84
```

## 共享映射文件

除了之前的`MAP_SHARED`选项，程序还可以调用`msync`函数将缓存区的改动直接写入磁盘文件。其前两个参数和`munmap`函数类似，可以选定内存区域；第三个参数可以选择：

* `MS_ASYNC`——异步更新改动。更新直接交由系统调度，比一定在函数返回之前就执行。
* `MS_SYNC`——同步更新改动。会立即更新，有可能被阻塞直到完成更新。不能和`MS_ASYNC`同时使用。
* `MS_INVALIDATE`——“All other file mappings are invalidated so that they can see the updated values.”撤销其他文件的映射，从而使当前的改动立即生效。

## `mmap`的其他用处

1. `mmap`可以用來执行文件的读写操作。相比于显式地文件I/O操作，`mmap`更快更方便。
2. 还可以在映射文件中构建数据结构(例如，struct结构体)。如果，重新映射文件，那么数据就会恢复到文件中那样。而以前的指针就会失效，除非这些指针都在原来的内存区域内而且文件被映射到原来的内存区域。
3. 还有一种特殊用法：将`/dev/zero`文件映射到内存中。这个文件像是内部填充了无限多个0字节。如果程序需要这样的文件，就可以映射这个`/dev/zero`文件。所有写入`/dev/zero`文件的内容都会被丢弃。有些内存分配程序也会通过映射`/dev/zero`来获取预初始化好的内存块。

# 4 管道

管道是一种单向通信机制。数据总是写入管道的“写入端”，并从“读取端”读取。管道是一种串行通信机制，数据的写入顺序也是其读取顺序。通常，管道用于同一进程两条线程之间的通信或者是父进程与子进程之间的通信。在Shell里，`|`可以创建管道。`ls | less`命令将`ls`命令的结果通过管道传输给`less`命令，作为其输入参数。

管道的容量是有限的。如果数据的写入速度比读取速度快，导致管道容量饱和，写入进程就会被阻塞；如果读取进程读不到数据也会被阻塞，直到有新数据被写入。所以，管道可以自动同步两条进程。

## 创建管道

C语言中可以使用`pipe`函数创建管道。其参数为一个大小为2的整数数组：待读取的文件描述符放在数组0位置，待写入的文件描述符放在数组1位置。


```C
int pipe_fds[2]; 
int read_fd;
int write_fd;

pipe (pipe_fds); 
read_fd = pipe_fds[0]; 
write_fd = pipe_fds[1];
```

上一代码中，数据从`read_fd`流向`write_fd`。

## 父子进程间的通信

一个进程的文件描述符不能传递给无关进程。只能通过fork函数创建子进程，由子进程继承父进程调用`pipe`函数产生的文件描述符。所以，管道只能连接相关进程。

```C
#include <stdlib.h> 
#include <stdio.h> 
#include <unistd.h>

/* Write COUNT copies of MESSAGE to STREAM, pausing for a second between each. */
void writer (const char* message, int count, FILE* stream) 
{
	for (; count > 0; --count) {
		/* Write the message to the stream, and send it off immediately. */ 
		fprintf (stream, “%s\n”, message);
		
		/* Everytime wirte sth into stream, execute FFLUSH immediately to send the msg*/
		fflush (stream); 
		
		/* Snooze a while. */
		sleep (1);
	} 
}

/* Read random strings from the stream as long as possible. */
void reader (FILE* stream) 
{
	char buffer[1024];
	/* Read until we hit the end of the stream. 
	 * fgets reads until either a newline or the end-of-file. 
	 */ 
	 while (!feof (stream)
			&& !ferror (stream)
			&& fgets (buffer, sizeof (buffer), stream) != NULL) 
		fputs (buffer, stdout);
}

int main () 
{
	int fds[2]; 
	pid_t pid;
	/* Create a pipe. File descriptors for the two ends of the pipe are placed in fds. */
	pipe (fds);

	/* Fork a child process. */ 
	pid = fork ();

	if (pid == (pid_t) 0) {
	
		FILE* stream;
		
		/* This is the child process. Close our copy of the write end 
		 * of the file descriptor. 
		 */
		close (fds[1]);
		
		/* Convert the read file descriptor to a FILE object, and read from it. */
		stream = fdopen (fds[0], “r”); 
		reader (stream);
		close (fds[0]);
	} else {
		/* This is the parent process. */
		FILE* stream;
		
		/* Close our copy of the read end of the file descriptor. */ 
		close (fds[0]);
		
		/* Convert the write file descriptor to a FILE object, and write to it. */
		stream = fdopen (fds[1], “w”); 
		writer (“Hello, world.”, 5, stream); 
		close (fds[1]);
	}

	return 0;
}
```
当Shell中执行`ls | less`命令时，less程序运行在ls程序fork出的子进程中。这样，子进程继承各自父进程的文件描述符，并通过管道通信。

## 重定向stdin、stdout、stderr

通常，程序需要改变子进程的输入输出，指向管道的一端。使用`dup2`函数即可实现这一目标。例如，要把进程的标准输入重定向到文件描述符`fd`上，可以通过这一代码来实现：

```C
dup2(fd, STDIN_FILENO);
```

代码示例。

```C
#include <stdio.h> 
#include <sys/types.h> 
#include <sys/wait.h> 
#include <unistd.h>

int main () 
{
	int fds[2]; 
	pid_t pid;
	
	/* Create a pipe. File descriptors for the two ends of the pipe are placed in fds. */
	pipe (fds);
	/* Fork a child process. */ 
	pid = fork ();

	if (pid == (pid_t) 0) {
		/* This is the child process. Close our copy of the write end of the file descriptor. */
		close (fds[1]);
		
		/* Connect the read end of the pipe to standard input. */ 
		dup2 (fds[0], STDIN_FILENO);
		
		/* Replace the child process with the “sort” program. */ 
		execlp (“sort”, “sort”, 0);
	} else {
		/* This is the parent process. */
		FILE* stream;
		
		/* Close our copy of the read end of the file descriptor. */ 
		close (fds[0]);
		
		/* Convert the write file descriptor to a FILE object, and write to it. */
		stream = fdopen (fds[1], “w”);
		fprintf (stream, “This is a test.\n”);
		fprintf (stream, “Hello, world.\n”);
		fprintf (stream, “My dog has fleas.\n”); 
		fprintf (stream, “This program is great.\n”);
		fprintf (stream, “One fish, two fish.\n”);
		fflush (stream);
		close (fds[1]);
		
		/* Wait for the child process to finish. */ 
		waitpid (pid, NULL, 0);
	}

	return 0;
}
```

## `popen`和`pclose`

之前的管道实现模式较为复杂，C语言中还可以使用`popen`和`pclose`实现同样的功能。

```C
#include <stdio.h>
#include <unistd.h>

int main () 
{
	FILE* stream = popen (“sort”, “w”);
	fprintf (stream, “This is a test.\n”); 
	fprintf (stream, “Hello, world.\n”);
	fprintf (stream, “My dog has fleas.\n”); 
	fprintf (stream, “This program is great.\n”); 	fprintf (stream, “One fish, two fish.\n”); 
	
	return pclose (stream);
}
```

上例中，`popen`函数创建了一条子进程来调用`sort`函数，第二个参数`"w"`表示要向子进程“写入”数据。函数的返回值是管道的一端。另一端与子进程的标准输入相连接。

`pclose`函数关闭子进程的管道，并会在子进程退出时返回其状态代码。

如果`popen`函数第二个参数为`"r"`，函数就会返回子进程的标准输出流以便父进程接收数据。

## FIFO

FIFO，也称作“命名管道”，可以连接任何(即使不相关的)进程；与之相对应，之前的管道也称作匿名管道。

在Shell中，可以调用`mkfifo`命令创建FIFO文件。

```shell
$ mkfifo /tmp/temp-fifo
$ ls -l /tmp/fifo

prw-rw-rw- 1 samuel users 0 Jan 16 14:04 /tmp/fifo
```
执行`ls`命令后，输出的第一个字符“p”表示该文件为FIFO管道文件。

在第一个窗口执行以下命令，作为管道的输出端。

```shell
$ cat < /tmp/fifo
```

在第二个窗口执行以下命令，作为管道的输入端。

```shell
$ cat > /tmp/fifo
```

这是在第二个窗口输入任何字符，按下ENTER键后，字符都会显示在第一个窗口中。

就像普通文件那样，执行rm命令就可以移除FIFO管道。

```shell
$ rm /tmp/fifo
```

### 创建FIFO管道

在C语言中，使用`mkfifo`函数创建FIFO管道文件。第一个参数是创建文件的路径；第二个参数是管道的访问权限。如果`mkfifo`函数创建失败，就会返回“-1”。

要调用`mkfifo`函数，必须引入`<sys/types.h>`和`<sys/stat.h>`两个头文件。

### 读写FIFO文件

在Linux中，FIFO管道就是个普通文件。程序要想通过FIFO管道通信，就必须打开文件来读写。可以使用`open`，` write`，`read`，`close`等低级I/O函数，也可以使用`fopen`，`fprintf`，`fscanf`，`fclose`等C语言标准函数。

```C
int fd = open (fifo_path, O_WRONLY); 
write (fd, data, data_length);
close (fd);
```

```C
FILE* fifo = fopen (fifo_path, “r”); 
fscanf (fifo, “%s”, buffer);
fclose (fifo);
```

一个管道文件可以被多个进程打开同时读写。每次写入或读取的数据最多不能超过PIPE_BUF(在Linux上为4KB)。

# 5 套接字

套接字机制允许不同电脑上的进程沟通。现实中，FTP、Telnet以及WWW都通过套接字机制来通信。

## 套接字的概念

一个完整的套接字包含三个部分：通信类型(communication style)、域名(namespace)、协议(protocal)。

套接字机制将要传送的数据封装为数据包(packet)。通信类型决定了如何传送数据包、如何标记收发者的地址：

* connection类型保证了数据包的接受顺序与发送顺序相同。如果数据包丢失，接受者会要求重新发送。在connection类型连接建立阶段，收发者的地址就明确了下来——有点想打电话，一旦拨通，就会把数据传送工作处理完毕。
* datagram类型并不保证收发顺序。每一个数据包都要标记接受者地址，但不保证一定能够送达。datagram类型有点像邮寄信件，每一次传送都要标记地址。

对于本地通信，套接字地址类似于普通文件名。对于Internet通信，套接字地址受IP协议约束。

协议指出了数据传输方式。可以是TCP/IP协议，可以是UNIX本地通信协议，也可以是某些组织的私有协议。

## 系统调用

套接字机制，相比前面四类方式，更为灵活。其相关的系统调用包括：

* `socket`，创建套接字；
* `close`，关闭套接字；
* `connect`，在两个套接字之间建立连接；
* `bind`，给套接字绑定地址；
* `listen`，配置套接字的监听状态；
* `accept`，接受连接，并创建新的套接字。

套接字由文件描述符来表示。

### 建立与关闭

如前所述，C语言中通过调用`socket`和`close`来创建和关闭套接字。创建套接字时应该指明：域名类型、通信类型、协议。

域名类型的常量都以`PF_`开头，即“protocal family”的简写。如，`PF_LOCAL`或`PF_UNIX`表示本地域名，`PF_INTE`表示互联网域名。通信类型的常量都以`SOCK_`开头，`SOCK_STREAM`和`SOCK_DGRAM`分别表示connection类型和datagram类型。协议是传输数据的低层机制。

`socket`调用成功后会返回一个文件描述符，可以对这个描述符执行`read`或`write`等操作。对于不再使用的套接字，调用`close`来移除套接字。

### 建立连接

客户端进程通过调用`connect`函数从本地套接字发起与服务器套接字(第二个参数)的连接。第三个参数是第二个参数所指向的结构体中地址的字节长度。

### 发送消息

和读写普通文件情形类似。

## 服务器

一个服务器的生命周期包括：建立connection类型套接字，调用`bind`绑定地址，调用`listen`监听连接请求，调用`accept`接受连接，关闭套接字。程序并不会直接通过服务器套接字读写数据，而是接受连接时新建一个独立的套接字来处理读写请求。

`bind`的第一个参数是指向服务器套接字的文件描述符，第二个参数是地址，第三个参数是地址的子节长度。

`listen`的第一个参数是服务器套接字的文件描述符，第二个参数是连接请求队列的长度。如果队列未饱和，后续请求就会排队等候服务器处理；如果队列饱和，后续请求将直接丢弃，不被受理。

`accept`的第一个参数是套接字文件描述符；第二个参数指向地址结构体，里面是客户端地址；第三个参数是套接字地址结构体的字节长度。调用`accept`会创建一个新的套接字，并返回相应的文件描述符。原来那个套接字保持监状态并建立新连接。

要从输入队列中读取但不移除数据，可以使用`recv`函数。其参数和`read`一致，但多了一个FLAGS选项，一定要填入`MSG_PEEK`。

## 本地套接字

同一电脑上的两个进程间通信使用**本地套接字**机制。

创建本地套接字时，域名类型要标记为`PF_LOCAL`或`PF_UNIX`。

套接字的名称标注在`sockaddr_un`结构体中。必须将`sun_family`域设置为`AF_LOCAL`，表示这是一个本地域名；`sun_path`域为文件名，不能超过108个字节。`sockaddr_un`结构体的长度应该用`SUN_LEN`宏来计算。

套接字创建进程必须具有目录写入权限，因为要生成套接字文件。要连接套接字的进程必须有读取权限。

要创建本地套接字，`socket`函数参数的协议域只能为0。

如果不再使用本地套接字，使用`unlink`来移除。

### 本地套接字机制的实现

服务器端 socket-server.c 文件。

```C
#include <stdio.h> 
#include <stdlib.h> 
#include <string.h> 
#include <sys/socket.h> 
#include <sys/un.h> 
#include <unistd.h>

/* Read text from the socket and print it out. 
 * Continue until the socket closes. Return nonzero 
 * if the client sent a “quit” message, zero otherwise. 
 */
int server (int client_socket) 
{
	while (1) { 
		int length; 
		char* text;
		
		/* First, read the length of the text message from the socket. 
		 * If read returns zero, the client closed the connection. 
		 */
		if (read (client_socket, &length, sizeof (length)) == 0) 
			return 0;
		
		/* Allocate a buffer to hold the text. */ 
		text = (char*) malloc (length);
		
		/* Read the text itself, and print it. */
		read (client_socket, text, length); 
		printf (“%s\n”, text);

		/* Free the buffer. */
		free (text);

		/* If the client sent the message “quit,” we’re all done. */ 
		if (!strcmp (text, “quit”))
			return 1; 
	}
}

int main (int argc, char* const argv[]) 
{
	const char* const socket_name = argv[1];
	int socket_fd;
	struct sockaddr_un name;
	int client_sent_quit_message;

	/* Create the socket. */
	socket_fd = socket (PF_LOCAL, SOCK_STREAM, 0); 
	
	/* Indicate that this is a server. */ 
	name.sun_family = AF_LOCAL;
	strcpy (name.sun_path, socket_name);
	bind (socket_fd, &name, SUN_LEN (&name));

	/* Listen for connections. */
	listen (socket_fd, 5);
	
	/* Repeatedly accept connections, spinning off one server() 
	 * to deal with each client. Continue until a client sends a “quit” message. 
	 */
	do {
		struct sockaddr_un client_name; 
		socklen_t client_name_len;
		int client_socket_fd;
		
		/* Accept a connection. */
		client_socket_fd = accept (socket_fd, &client_name, &client_name_len); 
		
		/* Handle the connection. */
		client_sent_quit_message = server (client_socket_fd);

		/* Close our end of the connection. */
		close (client_socket_fd);
	}
	
	while (!client_sent_quit_message);
	
	/* Remove the socket file. */ 
	close (socket_fd);
	unlink (socket_name);

	return 0;
}
```

客户端 socket-client.c 文件。

```C
#include <stdio.h> 
#include <string.h> 
#include <sys/socket.h> 
#include <sys/un.h> 
#include <unistd.h>

/* Write TEXT to the socket given by file descriptor SOCKET_FD. */
void write_text (int socket_fd, const char* text) 
{
	/* Write the number of bytes in the string, including NUL-termination. */
	int length = strlen (text) + 1;
	write (socket_fd, &length, sizeof (length)); 
	
	/* Write the string. */
	write (socket_fd, text, length);
}

int main (int argc, char* const argv[]) 
{
	const char* const socket_name = argv[1]; 
	const char* const message = argv[2];
	int socket_fd;
	struct sockaddr_un name;
	
	/* Create the socket. */
	socket_fd = socket (PF_LOCAL, SOCK_STREAM, 0);
	
	/* Store the server’s name in the socket address. */ 
	name.sun_family = AF_LOCAL;
	strcpy (name.sun_path, socket_name);

	/* Connect the socket. */
	connect (socket_fd, &name, SUN_LEN (&name));

	/* Write the text on the command line to the socket. */ 
	write_text (socket_fd, message);
	
	close (socket_fd);
	
	return 0;
}

```

客户端发送真正的文本消息之前，先发送消息的字节长度，通知服务器端先声明出足够大的缓存区域来存储消息。

上述代码编译成二进制可执行文件后，现在第一个窗口运行服务器端，指定`/tmp/socket`为套接字文件所在。

```shell
$ ./socket-server /tmp/socket
```
在另一个窗口运行客户端。

```shell
$ ./socket-client /tmp/socket “Hello, world.”
$ ./socket-client /tmp/socket “This is a test.”
```

如果要中止连接，运行

```shell
$ ./socket-client /tmp/socket “quit”
```

## 网络套接字

不同电脑上的两个进程间通信使用**网络套接字**机制。

创建网络套接字时，域名类型要标记为`PF_INET`。

网络套接字的地址标注在`sockaddr_in`结构体中。必须将`sin_family`域设置为`AF_INET`，表示这是一个本地域名；`sin_addr`域为32位IP地址。网络套接字地址包括IP地址和端口号两项。因为不同系统会讲多字节数据按不同的顺序存储，一定要用`htons`将端口数转换为网络字节顺序(network bytes order)。在Linux手册`ip`项查看详情。

可以使用`gethostname`函数将十进制IP地址转换为32位二进制格式。该函数返回一个`hostent`类型结构体的指针，其`h_addr`域包含了主机IP。

```C
#include <stdlib.h> 
#include <stdio.h> 
#include <netinet/in.h> 
#include <netdb.h> 
#include <sys/socket.h> 
#include <unistd.h> 
#include <string.h>

/* Print the contents of the home page for the server’s socket. 
 * Return an indication of success. 
 */
void get_home_page (int socket_fd) 
{
	char buffer[10000];
	ssize_t number_characters_read;
	
	/* Send the HTTP GET command for the home page. */ 
	sprintf (buffer, “GET /\n”);
	write (socket_fd, buffer, strlen (buffer));
	
	/* Read from the socket. The call to read may not return all the data at one time, 
	 * so keep trying until we run out. 
	 */ 
	 
	while (1) {
		number_characters_read = read (socket_fd, buffer, 10000); 
		if (number_characters_read == 0)
			return;
		/* Write the data to standard output. */
		fwrite (buffer, sizeof (char), number_characters_read, stdout);
	} 
}

int main (int argc, char* const argv[]) 
{
	int socket_fd;
	struct sockaddr_in name; 
	struct hostent* hostinfo;
	
	/* Create the socket. */
	socket_fd = socket (PF_INET, SOCK_STREAM, 0);
	/* Store the server’s name in the socket address. */ 
	name.sin_family = AF_INET;
	/* Convert from strings to numbers. */
	hostinfo = gethostbyname (argv[1]);

	if (hostinfo == NULL)
		return 1; 
	else
		name.sin_addr = *((struct in_addr *) hostinfo->h_addr); 
	
	/* Web servers use port 80. */
	name.sin_port = htons (80);
	/* Connect to the Web server */
	
	if (connect (socket_fd, &name, sizeof (struct sockaddr_in)) == -1) {
		perror (“connect”);
		return 1; 
	}

	/* Retrieve the server’s home page. */ get_home_page (socket_fd);

	return 0;
}
```

使用如下命令，来查看www.bing.com的网页：

```shell
$ ./socket-inet www.bing.com <html>

<meta http-equiv=”Content-Type” content=”text/html; charset=iso-8859-1”> 
...
```

## 套接字对

`sockerpair`函数可以给本地进程创建两个套接字文件描述符，允许进程双向通信。