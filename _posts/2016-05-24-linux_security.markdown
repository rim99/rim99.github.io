---
layout: post
title: Advanced Linux Programming - 系统安全
date: 2016-05-24 19:53:02
categories: 翻译
---


# 1 用户与组

每一个Linux用户都对应一个用户ID，或者UID。其对文件的读、写、执行等权限是一定的。也可以设置多个用户名对应同一个UID，那么这些用户就都完全共享权限。

如果有很多用户对某一类文件需要共享相同权限，可以把他们归为一个用户组里。每个组对应一个组ID，又叫GID。用户组里的成员只能是用户，不能是其他组。

调用`id`命令可以查看当前用户的ID信息。

超级用户的UID为0，用户名为“root”。超级用户具有最高权限。

```shell
# id
uid=0(root) gid=0(root) groups=0(root)
```

# 2 进程用户ID与进程组ID

进程的用户ID、组ID和进程的发起者相同。

可以在代码中查看：

```C
#include <sys/types.h> 
#include <unistd.h> 
#include <stdio.h>

int main() 
{
	uid_t uid = geteuid ();
	gid_t gid = getegid ();
	printf (“uid=%d gid=%d\n”, (int) uid, (int) gid); 
	
	return 0;
}
```

# 3 文件系统权限

Linux将用户对文件的操作定义为三种：读取、写入、执行。

使用`ls -l`命令可以查看目标文件的权限。

```shell
$ ls -l postgresql-9.5.2.tar.gz 

-rw-r--r-- 1 1000 staff 24100449 Mar 28 20:23 postgresql-9.5.2.tar.gz
```

上面命令的结果显示：文件所有人的UID为1（root用户），GID为1000；“所有人”具有读写权限，“组”和“其他”都有读取权限。

调用`chmod`可以改变文件的权限。

```shell
$ chmod o+w postgresql-9.5.2.tar.gz

$ ls -l postgresql-9.5.2.tar.gz

-rw-r--rw- 1 1000 staff 24100449 Mar 28 20:23 postgresql-9.5.2.tar.gz
```

如上所示，“其他”多了写入权限。

系统调用`stat`可以读取文件权限。第一个参数是文件路径，第二个参数是写入信息的`stat`类型结构体指针。

```C
#include <stdio.h>
#include <sys/stat.h>

int main (int argc, char* argv[]) 
{
	const char* const filename = argv[1]; 
	struct stat buf;

	/* Get file information. */
	stat (filename, &buf);
	
	/* If the permissions are set such that the file’s owner can write to it, print a message. */ 
	if (buf.st_mode & S_IWUSR)
		printf (“Owning user can write `%s’.\n”, filename); 
		
	return 0;
}
```

代码中`S_IWUSR`表示文件所有人的写入权限。类似的，`S_IRGRP`表示组的读取权限，`S_IXOTH`表示其他的执行权限。

需要注意的是，目录权限的意义和文件完全不同。

如果某个用户可以写入某个目录，那么他就可以删除目录下的任何文件，即使他对文件本身没有写入权限。如果他对某个目录有执行权限，那么他可以执行下面的任何一个文件。

在C语言中，可以使用`chmod`函数改变文件权限。

```C
chmod (“hello”, S_IRUSR | S_IXUSR);
```

## Sticky 位

如果目录设置了Sticky位，就不会出现上述越权删除文件的问题。

可以使用Shell命令：

```shell
chmod o+t directory
```

C语言可以通过设置`S_ISVTX`flag来实现。

```C
chmod (dir_path, S_IRWXU | S_IRWXG | S_IRWXO | S_ISVTX);
```

# 4 Real和Effective类ID

实际上，进程有两类ID：Real类、Effective类，UID和GID都是。通常，内核在意的是Effective类ID。但当进程要修改Effective类ID时，内核就会检查其Real类ID。

进程有时需要修改自身ID。

当服务器监听进程以root身份运行时，如果有用户A远程发起访问请求。内核必须检查A的权限，然后决定。如果，发起请求太多，内核就会忙于检查权限。实际上，进程会切换Effective类ID(root -> A)。以A的身份直接尝试请求访问。这样，内核可以处理得更快。

改变UID、GID的调用为`setreuid`、`setregid`。

基于安全考虑，在Linux中，只有root进程可以随意更改ID。其他进程只能执行三类操作：

* 将Effective类ID设置为Real类ID
* 将Real类ID设置为Effective类ID
* 将Effective类ID和Real类ID互换

如果不像保留Real类UID，可以执行：

```C
seteuid (id); 
setreuid (-1, id);
```

## setuid位

前面说了普通用户不能获取root身份，但是实际上，我们可以通过`sudo`来获取root权限。

这是因为，sudo实际上是一个setuid类程序。也就是当程序执行时，进程有效ID实际上是文件所有人。设置了setuid位的可执行文件都可以达到这个效果。

使用命令`chmond +s`或者代码中调用`S_ISUID`flag位都可以设置setuid位。

# 5 验证用户

如果你并不想让每个人都拥有setuid类程序的权限，就像`sudo`那样，需要密码通过验证才可以获取root权限。Linux提供了PAM模块(Pluggable Anthentication Modules)，可以做到这一点。

```C
#include <security/pam_appl.h> 
#include <security/pam_misc.h> 
#include <stdio.h>

int main () 
{
	pam_handle_t* pamh; 
	struct pam_conv pamc;
	
	/* Set up the PAM conversation. */
	pamc.conv = &misc_conv;
	pamc.appdata_ptr = NULL;
	
	/* Start a new authentication session. */ 
	pam_start (“su”, getenv (“USER”), &pamc, &pamh); 
	
	/* Authenticate the user. */
	if (pam_authenticate (pamh, 0) != PAM_SUCCESS) 
		fprintf (stderr, “Authentication failed!\n”);
	else
		fprintf (stderr, “Authentication OK.\n”);
	
	/* All done. */ 
	pam_end (pamh, 0); 
	
	return 0;
}
```

要编译上面这段代码pam.c，需要引入libpam和libpam_misc两个库。

```shell
$ gcc -o pam pam.c -lpam -lpam_misc
```

PAM的详细文档位于`/usr/doc/pam`。

# 6 一些安全漏洞

## 缓冲区溢出

缓冲区溢出攻击的本质是诱使程序执行一段第三方代码。通常的做法是在进程指令栈内写入代码。这样，当进程执行完当前指令，向栈内获取新指令时，就会得到攻击者写入的代码。如果执行进程有root权限，那么这个攻击的后果将是非常严重的。

假设一个获取用户名的网络程序代码如下所示：

```C
#include <stdio.h>
int main () 
{
	/* Nobody in their right mind would have more than 32 characters in their username. 
	 * Plus, I think UNIX allows only 8-character usernames.
	 * So, this should be plenty of space.
	 */  
	char username[32];
	
	/* Prompt the user for the username */
	printf (“Enter your username: “); 
	
	/* Read a line of input. */
	gets (username);
	
	/* Do other things here... */
	
	return 0;
}
```

如果攻击者在填写用户名时故意写入过长的字符串，多余的字符就会超出缓存区界限，污染周围的栈。

避免这类错误的办法就是尽量使用安全的高级调用，例如：

```C
char* username = getline (NULL, 0, stdin);
```

`getline`函数能够自动调用malloc获取足够大的内存区域。当然，使用完后一定要free掉，避免内存泄漏。

## `/tmp`的竞争条件

如果攻击者成功地将`/tmp`内文件链接到自有权限的文件上，那么他就可以从程序的临时读写中探嗅可用信息。

例如，一个prog程序要向`/tmp/prog`文件写入临时信息。如果攻击者提前建立了这个文件，并将这个文件链接到自己的文件。那么程序就会把数据写到攻击者所链接的文件上。

避免这一问题可方法之一是随机选择临时文件的名称；还有就是调用`open`打开文件时，传入`0_EXCL`标记。传入`0_EXCL`标记后，如果要打开的文件已经存在，`open`调用会立即失败。注意`0_EXCL`标记对于NFS文件系统是没有效果的。

下面代码给出了完整的解决方案，但并**不保证没有其他bug**。

```C
#include <fcntl.h> 
#include <stdlib.h> 
#include <sys/stat.h> 
#include <unistd.h>

/* Returns the file descriptor for a newly created temporary file. 
 * The temporary file will be readable and writable by the effective user ID of the current process 
 * but will not be readable or writable by anybody else.
 * Returns -1 if the temporary file could not be created. 
 */
int secure_temp_file () 
{
	/* This file descriptor points to /dev/random 
	 * and allows us to get a good source of random bits. 
	 */
	static int random_fd = -1;
	
	/* A random integer. */
	unsigned int random; 
	/* A buffer, used to convert from a numeric to a string representation of random. 
	* This buffer has fixed size, meaning that we potentially have a buffer overrun bug 
	* if the integers on this machine have a *lot* of bits. 
	*/
	char filename[128];

	/* The file descriptor for the new temporary file. */ 
	int fd;
	
	/* Information about the newly created file. */ 
	struct stat stat_buf;
	
	/* If we haven’t already opened /dev/random, do so now. 
	 * (This is not threadsafe.) 
	 */
	if (random_fd == -1) {
		/* Open /dev/random. Note that we’re assuming that 
		 * /dev/random really is a source of random bits, 
		 * not a file full of zeros placed there by an attacker. 
		 */
		random_fd = open (“/dev/random”, O_RDONLY);
	
		/* If we couldn’t open /dev/random, give up. */ 
		if (random_fd == -1)
			return -1; 
	}

	/* Read an integer from /dev/random. */
	if (read (random_fd, &random, sizeof (random)) !=
		sizeof (random)) return -1;
	
	/* Create a filename out of the random number. */ 
	sprintf (filename, “/tmp/%u”, random);
	
	/* Try to open the file. */
	fd = open (filename,
				/* Use O_EXECL, even though it doesn’t work under NFS. */ 
				O_RDWR | O_CREAT | O_EXCL,
				/* Make sure nobody else can read or write the file. */ 
				S_IRUSR | S_IWUSR);
	
	if (fd == -1) 
		return -1;
	
	/* Call lstat on the file, to make sure that it is not a symbolic link. */
	if (lstat (filename, &stat_buf) == -1) 
		return -1;

	/* If the file is not a regular file, someone has tried to trick us. */
	if (!S_ISREG (stat_buf.st_mode)) 
		return -1;

	/* If we don’t own the file, someone else might remove it, read it, or change it while we’re looking at it. */
	if (stat_buf.st_uid != geteuid () || stat_buf.st_gid != getegid ()) 
		return -1;
	/* If there are any more permission bits set on the file, something’s fishy. */
	if ((stat_buf.st_mode & ~(S_IRUSR | S_IWUSR)) != 0) 
		return -1;

	return fd;
}
```

## 使用`system`或`popen`

考虑一个服务器运行一个翻译程序。进程从客户端获取请求，然后根据请求，在本地查询词典，并将结果发送回去。

进程在查询词典时，可能调用命令：

```shell
$ grep -x word /usr/dict/words
```

看看程序是怎么调用这条命令的：

```C
#include <stdio.h>
#include <stdlib.h>

/* Returns a nonzero value if and only if the WORD appears in /usr/dict/words. */
int grep_for_word (const char* word) 
{
	size_t length; 
	char* buffer; 
	int exit_code;

	/* Build up the string ‘grep -x WORD /usr/dict/words’. 
	 * Allocate the string dynamically to avoid buffer overruns. 
	 */
	length = strlen (“grep -x “) + strlen (word) + strlen (“ /usr/dict/words”) + 1;
	buffer = (char*) malloc (length);
	sprintf (buffer, “grep -x %s /usr/dict/words”, word);

	/* Run the command. */
	exit_code = system (buffer);
	
	/* Free the buffer. */
	free (buffer);
	
	/* If grep returned 0, then the word was present in the dictionary. */ 
	return exit_code == 0;
}
```

如果攻击发送的查询字符是“`foo /dev/null; rm -rf /`”，那么问题就严重了。程序会执行命令：

```shell
$ grep -x foo /dev/null; rm -rf / /usr/dict/words
```

这实际上是两条命令。如果进程UID是0，第二条会递归删除根目录下的所有文件。`popen`函数也存在类似问题。

解决办法一是使用`exec`函数；二是解析请求，剔除分号一类的标点。




















