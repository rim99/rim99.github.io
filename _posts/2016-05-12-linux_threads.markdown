---
layout: post
title: Advanced Linux Programming - 线程
date: 2016-05-12 19:53:02
categories: 翻译
---

Linux使用标准的POSIX API——pthread来管理线程。所有pthread的函数与数据结构都声明在`<pthread.h>`中。pthread函数并不在标准C函数库中，而在libpthread中，所以在链接程序时要加上`-lpthread`选项。

# 1 线程的创建

进程中，每一个线程都一个线程ID。在C语言中，线程ID的类型是`pthread_t`。

创建出的线程只执行“线程函数”——一个包含了线程应该执行的代码的普通函数。函数返回时，线程也就退出。线程函数可以注入一个`void *`类型的线程参数，并返回`void *`类型值。返回值也可以用来在线程之间传值。

`pthread_create`函数可以用来创建线程。函数包括四个参数：

1. 一个`pthread_t`类型指针，用来储存新线程的ID。
2. 一个线程属性对象的指针。线程属性对象可以控制着新线程与其它部分的交互方式，也可以传入NULL。线程属性对象在后面还会讨论。
3. 一个`void *`类型线程函数指针。
4. 线程函数的参数，为`void *`类型。

调用`pthread_create`函数后可以立即创建新线程。Linux系统异步的调用各个线程，因此线程间的代码不能有顺序依赖。

除了线程函数正常返回外，线程还可以通过调用`pthread_exit`函数来结束线程。`pthread_exit`函数的参数就是线程的返回值。

## 给线程传值

`pthread_create`函数只能接受一个参数指针。如果要向参数传入多个参数，可以曲线救国——向函数传入一个结构体指针，在结构体中包含多个域。

需要注意的是：假设结构体参数在main函数中创建，如果main函数先于其他线程结束，那么结构体的内存空间就会被回收，而线程再次读写结构体的时候就会发生错误。

## 线程同步

上个问题的解决办法之一是让main函数等待其他线程结束后在返回。

这可以用`pthread_join`来实现。这个函数有两个参数：一个需要同步线程的ID，另一个是接收线程返回值的`void *`指针，也可以是NULL。

## 线程返回值

`pthread_join`函数接收线程返回值的是一个`void *`指针。如果要回传其它简单值，可以直接强制转换。原书P.67代码将`int *`类型与`void *`相互转换。

## 更多有关线程ID

有时会遇到需要检查当前线程的情形，例如线程不和自身同步，否则会回传`EDEADLK`错误。可以使用`pthread_self`来获取当前线程ID，使用`pthread_equal`来判定线程ID是否相同。

```C
if (!pthread_equal (pthread_self (), other_thread)) 
	pthread_join (other_thread, NULL);
```

## 线程属性

线程分为可同步(joinable)和可分离(detachable)两种。可同步线程在中止后需要系统回收资源；可分离线程在结束后，系统自动回收其资源。

分离线程的一个方法是调用`int pthread_detach(pthread_t tid)`函数。如果想要创建一个分离线程，就可以设置线程属性参数。

```C
#include <pthread.h>

void* thread_function (void* thread_arg) 
{
	/* Do work here... */ 
}

int main () 
{
	pthread_t thread;
	
	/* create new thread attribute object */
	pthread_attr_t attr;
	pthread_attr_init (&attr);
	pthread_attr_setdetachstate (&attr, PTHREAD_CREATE_DETACHED);
	
	/* create new thread */
	pthread_create (&thread, &attr, &thread_function, NULL);
	pthread_attr_destroy (&attr);
	
	/* Do work here... */
	/* No need to join the second thread. */
	
	return 0;
}
```

# 2 线程注销

线程可以正常的自行退出，也可以有另一线程请求注销(Cancellation)。可以调用`pthread_cancel`函数来注销线程。注销后的线程应该被同步，因为他的资源还没有被回收。当然了，分离线程注销后不用同步。注销线程的返回值是`PTHREAD_CANCLED`。

通常情况下，如果线程在执行过程中被注销，其资源无法回收，造成内存泄漏。线程的注销类型可以分为以下三种情况：

1. 如果线程可以在执行过程被立即注销，那就是*可异步注销*。
2. 如果线程可以在执行过程收到注销请求后，不能立即注销。请求进入队列，等到合适的时候，线程才注销，那就是*可同步注销*。这些合适的注销时间点就叫做注销点(Cancellation point)。
3. 如果线程总是忽略注销请求，那就是*不可注销*。

## 同步线程与异步线程

要设定一条线程为异步类型，可以使用`pthread_setcanceltype`函数，第一个参数应为`PTHREAD_CANCEL_ASYNCHRONOUS`或`PTHREAD_CANCEL_DEFERRED`。第二个参数为一个可NULL指针，用来存储该线程之前的注销类型。

`pthread_testcancel`函数可以用来设置注销点。该函数只处理可同步注销线程的挂起的注销请求。

还有其它注销类函数都可以在Linux手册页pthread_cancel项里找到。

## 不可注销区域

`pthread_setcancelstate`函数可以设定线程是否可注销——第一个参数设定为`PTHREAD_CANCEL_DISABLE`，线程就无法注销，设定为`PTHREAD_CANCEL_ENABLE`则有可以注销。第二个参数仍然为一个可NULL指针，用来存储该线程之前的注销类型。

利用`pthread_setcancelstate`函数可以设置一个关键的代码段——要么执行全段代码，要么一行都不执行。

## 何时注销线程

通常并不强制线程注销，而是通知对方在合适的时候自行退出。

# 3 线程独有数据

线程之间共享着同一进程的全局数据。其各自也有相互独立的数据存储空间。GNU/Linux给每个线程都分配有线程独有数据(thread-specific data)区。由于每个线程都共享同样的内存空间，因此常规方法无法读取线程独有数据区，而需要特殊的函数。

线程独有数据都是`void *`类型，可以创建无数多个。每项线程独有数据都需要一个key来对应。可以使用`pthread_key_create`函数来创建key。`pthread_key_create`函数的第一个参数是一个`pthread_key_t`类型的变量指针。第二个参数是一个清尾函数(cleanup function)指针，指针所指函数可以在线程退出时执行，同时传递key所对应的值。清尾函数也会在线程被注销时调用。需要注意的是：如果key所对应的值为NULL，那么清尾函数不会执行。如果不需要清尾函数，就将第二个参数设为NULL就行。

key创建好以后，可以使用`pthread_setspecific`函数来设定值。第一个参数是key，第二个参数是要存储的`void *`类型值。要根据key来读值，可以使用`pthread_getspecific`函数。

下面（原书P.73-74）代码展示了线程独有数据区的用法。

```C
#include <malloc.h> 
#include <pthread.h> 
#include <stdio.h>

/* The key used to associate a log file pointer with each thread. */ 
static pthread_key_t thread_log_key;

/* Write MESSAGE to the log file for the current thread. */
void write_to_thread_log (const char* message) 
{
	FILE* thread_log = (FILE*) pthread_getspecific (thread_log_key);
	fprintf (thread_log, “%s\n”, message); 
}

/* Close the log file pointer THREAD_LOG. */ 
void close_thread_log (void* thread_log)
{
	fclose ((FILE*) thread_log);
}

void* thread_function (void* args) 
{
	char thread_log_filename[20]; 
	FILE* thread_log;
	
	/* Generate the filename for this thread’s log file. */
	sprintf (thread_log_filename, “thread%d.log”, (int) pthread_self());

	/* Open the log file. */
	thread_log = fopen (thread_log_filename, “w”);

	/* Store the file pointer in thread-specific data under thread_log_key. */ 
	pthread_setspecific (thread_log_key, thread_log);
	
	write_to_thread_log (“Thread starting.”); 
	/* Do work here... */

	return NULL; 
}

int main () 
{
	int i;
	pthread_t threads[5];
	/* Create a key to associate thread log file pointers in thread-specific data. 
	Use close_thread_log to clean up the file pointers. */
	pthread_key_create (&thread_log_key, close_thread_log); 
	
	/* Create threads to do the work. */
	for (i = 0; i < 5; ++i)
		pthread_create (&(threads[i]), NULL, thread_function, NULL); 
		
	/* Wait for all threads to finish. */
	for (i = 0; i < 5; ++i)
		pthread_join (threads[i], NULL); 
		
	return 0;
}
```

注意`thread_function`中不需要关闭文件描述符，因为main函数中

```C
pthread_key_create (&thread_log_key, close_thread_log);
```

已经设定了清尾函数来执行这个关闭动作。

## 清尾句柄

清尾函数有助于确保线程资源不会泄漏，但只能在设置有线程独有数据区的时候才能用。Linux提供了清尾句柄(cleanup handler)可以突破这一限制。

清尾句柄函数可在线程退出时调用。句柄参数为一个`void *`类型值，并在句柄注册时提供，因此可以在多个线程使用同一个清尾句柄函数。

清尾句柄函数是个临时措施。只能在线程执行一段代码后退出或注销时调用；如果线程没有退出或注销，资源需要显式的回收，清尾句柄应该被注销。

>A cleanup handler is a temporary measure, used to deallocate a resource only if the thread exits or is canceled instead of finishing execution of a particular region of code. Under normal circumstances, when the thread does not exit and is not canceled, the resource should be deallocated explicitly and the cleanup handler should be removed.

使用`pthread_cleanup_push`函数来注册句柄。其参数为清尾函数指针和一个`void *`类型参数值。`pthread_cleanup_push`函数与`pthread_cleanup_pop`函数一一对应。后者可以清除句柄。如果向后者传入非零参数值，就可以执行并注销句柄。

## C++中的清尾句柄

略。原书P.76－77。

# 4 同步与关键区

## 竞争条件

假设一个程序利用若干线程并发执行链表中的作业。每当线程执行完当前作业时，就检查作业链表，如果非空，就执行表头作业，并将表头指针指向下一个作业。

如果，有两条线程同时结束执行，而作业链表中只有一个待选作业。假设其中的线程A检查完链表后选定了仅有的作业时，Linux系统突然挂起线程A，选择执行线程B。而线程B执行了同样的动作。这时问题出现了：两条线程同时执行同一作业。

更坏的情形是：线程A移动了表头指针后被挂起，线程B去检查作业链表时，执行`job_queue -> next`会发生段错误。

为了避免竞争条件，需要将某些操作**原子化**。原子操作是不可分割、不被打扰的：一旦开始，不能暂停或中止；同时也不能执行其他作业，直至原子操作结束。

## 互斥锁

解决作业队列竞争条件的方案是只允许一条线程评估作业队列。如果已经有一条线程在评估作业队列，那么直到该线程评估结束乃至移动表头指针之前，其它线程都只能等待。

Linux提供了Mutexes机制可以实现这一方案。Mutexes是*MUTual EXclusion locks*的简写，也就是互斥锁。互斥锁只能同时被一条线程锁定，其余线程尝试锁定互斥锁的时候，会被阻塞直到第一条线程结束锁定。

要创建互斥锁，首先要新建一个`pthread_mutex_t`类型值，并将其指针传递给函数`pthread_mutex_init`。函数的第二个参数是一个互斥锁属性对象指针，如果为NULL，就会使用默认属性。`pthread_mutex_t`类型变量只能初始化一次。

另一个创建互斥锁的简单方法是将`pthread_mutex_t`类型值直接赋值为`PTHREAD_MUTEX_INITIALIZER`，无需调用`pthread_mutex_init`函数。

线程可以调用`pthread_mutex_lock`和`pthread_mutex_unlock`锁定或解锁互斥锁。

```C
#include <malloc.h>
#include <pthread.h>

struct job {
	/* Link field for linked list. */ 
	struct job* next;

	/* Other fields describing work to be done... */ 
};

/* A linked list of pending jobs. */ 
struct job* job_queue;

/* A mutex protecting job_queue. */
pthread_mutex_t job_queue_mutex = PTHREAD_MUTEX_INITIALIZER;

/* Process queued jobs until the queue is empty. */

void* thread_function (void* arg) {

	while (1) {
		
		struct job* next_job;
		
		/* Lock the mutex on the job queue. */ 
		pthread_mutex_lock (&job_queue_mutex);
		
		/* Now it’s safe to check if the queue is empty. */ 
		if (job_queue == NULL)
			next_job = NULL; 
		else {
			/* Get the next available job. */
			next_job = job_queue;

			/* Remove this job from the list. */
			job_queue = job_queue->next;
		}
		
		/* Unlock the mutex on the job queue 
		because we’re done with the queue for now. */ 
		pthread_mutex_unlock (&job_queue_mutex);
		
		/* Was the queue empty? If so, end the thread. */ 
		if (next_job == NULL)
			break;
			
		/* Carry out the work. */ 
		process_job (next_job);
		
		/* Clean up. */
		free (next_job);
	}

	return NULL;
}
```

需要注意的是，如果线程在执行互斥锁内代码的中途退出，那么互斥锁将被永久锁定。

## 互斥死锁

互斥锁是一条线程拥有阻塞其它线程的能力，从而产生了新的bug类型——死锁。当一条线程阻塞在一件永运不会发生的事情上，就会产生死锁。

一种简单的死锁情形是——一条线程尝试锁定已被本线程锁定了的互斥锁：

* 对于快速型互斥锁(Fast Mutex)，也就是默认类型：本线程会被永久阻塞。
* 对于递归互斥锁(Recursive Mutex)，本线程不会阻塞。互斥锁会统计被线程锁定的次数，并要求同样次数的解锁操作才能完全解锁。
* 对于检错互斥锁(Error-checking Mutex)，第二次使用`pthread_mutex_lock`锁定会返回`EDEADLK`错误。

向`pthread_mutex_init`第二个参数传入`pthread_mutexattr_t`型指针可以设置互斥锁的类型。`pthread_mutexattr_t`类型值可以使用`pthread_mutexattr_setkind_np`函数设为`PTHREAD_MUTEX_RECURSIVE_NP`或`PTHREAD_MUTEX_ERRORCHECK_NP`，分别对应递归互斥锁和检错互斥锁。

```C
pthread_mutexattr_t attr; 
pthread_mutex_t mutex;

pthread_mutexattr_init (&attr);
pthread_mutexattr_setkind_np (&attr, PTHREAD_MUTEX_ERRORCHECK_NP);
pthread_mutex_init (&mutex, &attr);
pthread_mutexattr_destroy (&attr);
```

## 非阻塞互斥锁测试

Linux提供了`pthread_mutex_trylock`函数可以非阻塞的测试互斥锁是否锁定。如果没有锁定，函数返回0；如果锁定，函数返回`EBUSY`错误码。

## 信号量

在前述情形中，如果新作业入队速度跟不上线程的处理速度，线程就会在某一时刻检查到队列为空然后退出。而后续新作业入队后却发现没有线程来执行。这时就需要一种阻塞线程的机制。

**信号量**机制就是利用计数器来同步多条线程。有了互斥锁的配合，信号量的操作都可以原子化。

信号量的计数总是非负的。信号量支持两类基本操作：

1. `wait`操作可以将信号量减1。如果信号量为0，该操作就会被阻塞直到信号量为正。
2. `post`操作可将信号量加1。如果信号量为0，而且有`wait`操作被阻塞，那么此时其中一条线程的`wait`操作返回，线程得以继续执行。

Linux提供两类信号量机制：如果是线程间的信号量，直接使用POSIX API；如果是进程间的信号量机制，下一章有讲解。

要使用信号量，必须在代码中引用`<semaphore.h>`头文件。

信号量是一个`sem_t`类型变量，可以使用`sem_init`函数来初始化。

如果不再使用信号量，应该用`sem_destroy`函数来回收。

要执行`wait`操作可以调用`sem_wait`函数，要执行`post`操作可以调用`sem_post`函数。如果想要非阻塞的执行`wait`操作可以调用`sem_trywait`函数：如果信号量为0，函数就会立刻返回`EAGAIN`错误代码。

使用`sem_getvalue`函数可以获得当前信号量的值。

```C
#include <malloc.h> 
#include <pthread.h> 
#include <semaphore.h>

struct job {
	/* Link field for linked list. */ 
	struct job* next;

	/* Other fields describing work to be done... */ 
};

/* A linked list of pending jobs. */ 
struct job* job_queue;

/* A mutex protecting job_queue. */
pthread_mutex_t job_queue_mutex = PTHREAD_MUTEX_INITIALIZER;

/* A semaphore counting the number of jobs in the queue. */
sem_t job_queue_count;

/* Perform one-time initialization of the job queue. */
void initialize_job_queue () 
{
	/* The queue is initially empty. */
	job_queue = NULL;
	
	/* Initialize the semaphore which counts jobs in the queue. 
	 * Its initial value should be zero. 
	 */ 
	sem_init (&job_queue_count, 0, 0);
}

/* Process queued jobs until the queue is empty. */
void* thread_function (void* arg) 
{
	while (1) {
		struct job* next_job;
		
		/* Wait on the job queue semaphore. If its value is positive, 
		 * indicating that the queue is not empty, decrement the count by 1.
		 * If the queue is empty, block until a new job is enqueued. */
		sem_wait (&job_queue_count);

		/* Lock the mutex on the job queue. */
		pthread_mutex_lock (&job_queue_mutex);
		
		/* Because of the semaphore, we know the queue is not empty. 
		 * Get the next available job. */
		next_job = job_queue;

		/* Remove this job from the list. */
		job_queue = job_queue->next;

		/* Unlock the mutex on the job queue 
		 * because we’re done with the queue for now. 
		 */ 
		 pthread_mutex_unlock (&job_queue_mutex);
		
		/* Carry out the work. */ 
		process_job (next_job);

		/* Clean up. */
		free (next_job);
	}

	return NULL; 
}

/* Add a new job to the front of the job queue. */
void enqueue_job (/* Pass job-specific data here... */) 
{
	struct job* new_job;
	
	/* Allocate a new job object. */
	new_job = (struct job*) malloc (sizeof (struct job)); 
	
	/* Set the other fields of the job struct here... */

	/* Lock the mutex on the job queue before accessing it. */ 
	pthread_mutex_lock (&job_queue_mutex);

	/* Place the new job at the head of the queue. */ 
	new_job->next = job_queue;
	job_queue = new_job;

	/* Post to the semaphore to indicate that another job is available. 
	 * If threads are blocked, waiting on the semaphore, 
	 * one will become unblocked so it can process the job. 
	 */
	sem_post (&job_queue_count);

	/* Unlock the job queue mutex. */
	pthread_mutex_unlock (&job_queue_mutex);
}
```

上面代码中，线程每次从队列中获取作业任务时，先在信号量上减1。如果信号量为0，也就是队列中没有任务，线程就会被阻塞。

## 条件变量

假设你写了一个线程函数，来无限循环执行某些操作。而这一循环受变量flag的控制：如果flag被设定为真，循环执行；如果flag为假，则循环暂停。

通常的做法是，使用轮询模式原子地检查flag的设置。但是轮询模式在flag长时间为假时，会浪费大量的CPU资源执行开关互斥锁、检查flag设置等操作。而我们正真需要的是当条件不满足时，线程休眠直到条件得到满足。

Linux提供了**条件变量(Condition Variable)**机制。该机制可以预设线程执行的条件和休眠的条件。

如果有多条线程的执行条件相同，那么当条件为真发出信号时，只能启动第一个等待信号的线程。其余线程继续排队等待下一次信号。如果没有线程排队，则这一次的信号就直接消失而不会等候线程阻塞。

假设第一条线程检查flag为假，但在等候之前被挂起。此时如果有第二条线程改动flag并发出信号量时，第一条线程尚未进入等候状态，因此没有接收到信号，从而有可能永远被阻塞。

要避免这一竞争条件，必须使用互斥锁将*检查flag设置*与*等候信号量两个操作*原子化：每次循环先加锁，然后检查flag设置，如果条件为真则解锁并继续执行，如果条件为假，则将解锁和等候两个操作原子化执行。

C语言中，条件变量变量为一个`pthread_cond_t`类型变量。记得同时再声明一个互斥锁。

条件变量使用`pthread_cond_init`函数来初始化。

函数`pthread_cond_signal`用来发出信号。调用`pthread_cond_broadcast`可以将所有等候线程解除阻塞。

`pthread_cond_wait`可以阻塞调用线程，第一个参数是`pthread_cond_t`类型指针，第二个参数是`pthread_mutex_t`类型指针。调用`pthread_cond_wait`函数时，必须确保`pthread_mutex_t`指针所指互斥锁已经被锁定。

每次条件变动时，程序都会执行些列动作：

1. 锁定互斥锁；
2. 改变条件；
3. 发出信号，signal或broadcast；
4. 解锁互斥锁。

```C
#include <pthread.h>

int thread_flag;
pthread_cond_t thread_flag_cv; 
pthread_mutex_t thread_flag_mutex;

void initialize_flag () 
{
	/* Initialize the mutex and condition variable. */ 
	pthread_mutex_init (&thread_flag_mutex, NULL); 
	pthread_cond_init (&thread_flag_cv, NULL);

	/* Initialize the flag value. */
	thread_flag = 0; 	
}

/* Calls do_work repeatedly while the thread flag is set; 
 * blocks if the flag is clear. 
 */
void* thread_function (void* thread_arg) 
{
	/* Loop infinitely. */ 
	while (1) {
		/* Lock the mutex before accessing the flag value. */ 
		pthread_mutex_lock (&thread_flag_mutex);
		
		while (!thread_flag)
			/* The flag is clear. Wait for a signal on the condition variable,
			 * indicating that the flag value has changed. When the signal arrives 
			 * and this thread unblocks, loop and check the flag again. 
			 */
			pthread_cond_wait (&thread_flag_cv, &thread_flag_mutex);
	
		/* When we’ve gotten here, we know the flag must be set. 
		 * Unlock the mutex. 
		 */
		pthread_mutex_unlock (&thread_flag_mutex); 
			
		/* Do some work. */
		do_work ();
	}
		
	return NULL; 
}

/* Sets the value of the thread flag to FLAG_VALUE. */
void set_thread_flag (int flag_value) 
{
	/* Lock the mutex before accessing the flag value. */ 
	pthread_mutex_lock (&thread_flag_mutex);

	/* Set the flag value, and then signal in case thread_function 
	 * is blocked, waiting for the flag to become set. 
	 * However, thread_function can’t actually check the flag 
	 * until the mutex is unlocked. 
	 */
    thread_flag = flag_value; 
    pthread_cond_signal (&thread_flag_cv);
    
	/* Unlock the mutex. */ 
	pthread_mutex_unlock (&thread_flag_mutex);
}

```

## 两个或更多线程之间的死锁

不同线程间也有可能形成死锁。试考虑，线程A正在等候线程B变动条件变量，发出信号。与此同时，线程B在等候线程A变动条件变量，发出信号。这是两个线程就会永远阻塞，形成死锁。

还以一种可能，一条线程锁定了一组互斥锁，如果另一条线程以不同的顺序试图加锁同一组互斥锁，也会形成死锁。

# 5 线程在Linux系统的实现

Linux中，当程序调用`pthread_create`函数创建线程时，Linux会给新线程分配一个进程。使用`ps`命令可以查看到各个线程。

## 信号处理

如果多线程程序接收到一个信号，将由哪一条线程来处理呢？

不同的Unix系统有不同的处理方式。在Linux中，线程以进程的形式来实现。信号总会按照进程ID来发送。因此，不会有任何分歧。

## `clone`系统调用

无论是进程的创建`fork`还是线程的创建`pthread_create`，都统一于Linux系统调用`call`函数。该函数可以根据调用者的需要，来明确原进程与新进程之间共享的资源。

一般情况下，不推荐调用`clone`函数。

# 6 进程 v.s. 线程

* 程序的子线程必须运行同样的程序；而子进程可以运行其他程序。
* 子线程出错会危及到其它线程，因为它们之间有共享的内存空间；而子进程是独立的拷贝，因此不会因此出错。
* 创建进程时拷贝内存空间，会造成巨大的性能开销。不过只有内存发生变动时才会实际执行拷贝动作。因此如果子进程只读取内存就不会造成性能开销。
* 并行度较细时应该使用线程，较粗时使用进程。*并行度较细*是指并行任务的相似程度。
* 线程之间共享内存，容易出错，例如之前锁锁的竞争条件问题。而进程之间使用IPC机制来通信，bug较少，但是性能开销较大。


































