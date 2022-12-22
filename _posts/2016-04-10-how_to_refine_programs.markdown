---
layout: post
title: "CSAPP学习笔记 - 如何优化程序"
date: 2016-04-10 19:53:02 +8000
categories: 笔记
---



>本文为CSAPP第五章学习笔记。
 
编写高效的程序需要：

1. 选择合适的数据结构和算法
2. 编写出编译器能够有效优化以转换成高效可执行代码的源代码
3. 对于计算量较大的任务，可以将其分解为若干小的代码段，然后并行计算
 
 

 
 
优化代码：

1. 减少不必要的内容，让代码尽可能简单的执行期望的工作。如消除不必要的函数调用、条件测试和存储器引用。
2. 利用处理器提供的**指令集并行**能力，同时执行多条指令。根据代码的各项操作的时序特性做出合理安排，以避免不宜要的等待。


在优化代码的时候，要保证代码的简洁和可读性，因为代码终归需要维护和扩展。


# 1 减少存储器调用

考虑如下两个函数：

```C
void twiddle1(int *xp, int *yp)
{
    *xp += *yp;
    *xp += *yp; 
}

void twiddle2(int *xp, int *yp)
{
    *xp += 2* *yp;
}
```

`twiddlw1()`和`twiddle2()`有着相似的计算行为。但是，`twiddle2()`只有三次寄存器调用：`*xp`的读取，`*yp`的读取以及`*xp`的写入。`twiddle1()`则有六次寄存器调用，实际计算更为麻烦。

`twiddlw1()`和`twiddle2()`也不完全相同。试考虑，`*xp = *yp`，`twiddlw1()`的`*xp`实际等于`4* *xp`，而`twiddlw2()`的`*xp`实际等于`3* *xp`。这种*两个指针指向统一存储器*的情况，也称作*存储器别名使用*。

# 2 减少函数调用

考虑如下两个函数：

```C
void cycle1(int *xp, int *yp)
{
    return f() + f() + f() + f();
}

void cycle2(int *xp, int *yp)
{
    return 4*f();
}
```

`cycle1()`函数调用了`f()`函数四次，需要在寄存器执行4次 *建立栈帧 -> 计算f() -> 恢复栈帧* 过程。而`cycle2()`函数只需要一次。

当然，作为特殊情况，需要考虑`f()`会改变全局变量的可能。如果确实改变，那么`cycle1()`函数就会改变全局变量4次，而`cycle2()`函数只会改变1次。

# 3 每元素周期数CPE

考虑一个计算向量的前置和(prefix sum)的过程。

>向量P=(a1, a2, a3, .., ai,..., an)的前置和p(i)的计算过程可写为:
>
	```python
	p(1) = a1;
	p(i) = p(i-1) + ai; # i > 1
	```

*代码1*

```C
void combine(vec_pointer ptr, data_t *dest)
{
    long int i;                     
    *dest = 0;
    for (i = 0; i < length(ptr); i++) { /* length(ptr)返回得是ptr所指向的向量的所含元素个数  */
        data_t val;                     /* 新建一个内存空间 */
        get_vec_element(ptr, i, &val);  /* 将向量指针ptr所指的向量的第i项存入val */
        *dest = *dest + val;            /* *dest累加val，求的前置和 */
    }
}
```

代码1使用for循环迭代计算前置和。无论ptr所指向量有多大，每次调用函数都会执行*建立栈帧*和*恢复栈帧*，这一段代码的耗时是一个定值S。随着循环次数n的变化，总的耗时T＝S + n*L，其中L为for循环的单位循环执行时间，在本书中又称作**每元素周期数CPE**。

# 4 消除循环的低效率——代码移动

代码1在for循环中，每次循环都会调用`length()`获取向量的所含元素个数。然而这个值通常都是一个定值，如果将其存储在一个局部变量中，降低调用频率，可以有效改善代码运行效率。

*代码2*

```C
void combine(vec_pointer ptr, data_t *dest)
{
    long int i;                     
    *dest = 0;
    long int v_length = length(ptr);   /* 将length(ptr)返回值存储在局部变量中 */
    for (i = 0; i < v_length; i++) { 
        data_t val;                     
        get_vec_element(ptr, i, &val);  
        *dest = *dest + val;            
    }
}
```

# 5 减少过程调用

代码2的for循环中，每次都要掉用`get_vec_element()`来获得第i位元素，也同样代价巨大。一个合理的替代方案是：直接获取向量，存入局部变量中，然后按需调用。

*代码3*

```C
/*
 * 已知vec_pointer结构体的定义 
 */
typedef struct  {
    long int len;
    data_t *data;
} vec_rec, vec_pointer;

/*
 * 新建函数
 */
data_t *get_vec_start(vec_pointer ptr)
{
    return ptr->data;                   /* 直接获得ptr所指向量的数据部分的头段指针 */
}

void combine(vec_pointer ptr, data_t *dest)
{
    long int i;                     
    *dest = 0;
    long int v_length = length(ptr);    
    data_t *data = get_vec_start(ptr);  /* 将ptr数据存入数组 */
    for (i = 0; i < v_length; i++) { 
        *dest = *dest + data[i];        
    }
}
```

# 6 消除不必要的存储器引用

下面是代码3中for循环的汇编代码：

```Assembly
/*
 * code3: data_t = float
 * i 位于 %rdx, data 在 %rax, dest 在 %rbp, 越界标志 limit 在 %r12
 */

.L498:
    movss (%rbp), %xmm0           /* 取出dest，存入 %xmm0 */
    mulss (%rax, %rdx, 4), %xmm0  /* 取出data[i], 并与dest相乘 */
    movss %xmm0, (%rbp)           /* 将结果存入dest */
    addq  $1, %rdx                /* i加一 */
    cmpq  %rdx, %r12              /* 比较i是否越界 */
    jg    .L498                   /* 如果没有越界，就再次循环 */
```

可以看出代码3的for循环中，每次计算加法都会先引用寄存器中*dest所指向的空间，然后加和，最后将计算结果存入寄存器。但比较浪费的是，每次读取的值都是上次的计算结果。

合理的解决办法是，将累加值存入局部变量中，当计算结束后再把最终结果存入寄存器。

*代码4*

```C
void combine(vec_pointer ptr, data_t *dest)
{
    long int i; 
    long int v_length = length(ptr);    
    data_t *data = get_vec_start(ptr);  
    data_t acc = 0;                     /* 局部变量存储累加值 */
    for (i = 0; i < v_length; i++) { 
        acc = acc + data[i];            /* 用累加器acc累加data[i]，求前置和 */
    }
    *dest = acc;                        /* 将结果存入寄存器 */
}
```

其对应汇编代码

```Assembly
/*
 * code4: data_t = float
 * i 位于 %rdx, data 在 %rax, 越界标志 limit 在 %rbp, acc 在 %xmm0
 */

.L488:
    mulss (%rax, %rdx, 4), %xmm0  /* 取出data[i], 并与acc相乘 */
    addq  $1, %rdx                /* i加一 */
    cmpq  %rdx, %rbp              /* 比较i是否越界 */
    jg    .L488                   /* 如果没有越界，就再次循环 */
```

# 7 循环展开

至此，for循环内部代码已经足够简洁。然而，循环本身也存在开销，如果能够在保证计算结果足够精准的情况下，减少循环次数，也能产生明显的改善效果。

**循环展开**，就是一种程序变换，*通过增加每次迭代计算的元素的数量，来减少循环的迭代次数*。循环展开从两方面改善了程序的性能：

1. 减少了不直接有助于程序结果的操作的数量，如循环索引计算、条件分支；
2. 可以进一步变化代码，减少整个计算的关键路径上的操作数量。
    
>关键路径：在循环的反复执行过程中形成的数据相关链。    

*代码5*

```C
void combine(vec_pointer ptr, data_t *dest)
{
    long int i; 
    long int v_length = length(ptr);         
    long int limit = v_length - 1;      
    data_t *data = get_vec_start(ptr);       
    data_t acc = 0;                          
    
    /*循环1*/
    for (i = 0; i < limit; i+=2) {           /* 步进为2 */
        acc = (acc + data[i]) + data[i+1];   /* acc累加下两个data[i]，求前置和 */
    }
    
    /*循环2*/
    for (; i < length; i++) {                /* 累加剩余元素 */
        acc = acc + data[i];
    }
    *dest = acc;                             
}
```

观察代码5，有两个for循环：

1. 对于第一个循环，要保证循环不会越界(特别是`data[i+1]`)；
2. 要保证当循环索引`i`满足`i<n-1`实才会执行循环，因此最大索引`i+1`满足`i+1<(n-1)+1=n`。

# 8 提高并行性

观察代码4的循环中的计算：每次计算acc之前必须等前一循环的acc计算完成后才能继续。

同样代码5中循环1的计算行：每次计算acc之前先算`acc + data[i]`，然后计算`+ data[i+1]`，同样需要等待前已循环的acc计算完毕。

也就是说，acc的计算构成了一个单序列的计算流程，也就是一条关键路径。如果能够将这个流程拆分，就可以利用CPU的*乱序*特性，同时计算，提高效率。

一个可行的方法就是：先分别计算奇数位元素、偶数位元素的和，然后将两者加和。

*代码6*

```C
void combine(vec_pointer ptr, data_t *dest)
{
    long int i; 
    long int v_length = length(ptr);         
    long int limit = v_length - 1;      
    data_t *data = get_vec_start(ptr);       
    data_t acc1 = 0;
    data_t acc2 = 0;                          
    
    /*循环1*/
    for (i = 0; i < limit; i+=2) {           
        acc1 = acc1 + data[i];     /* 仅奇数位元素求前置和*/
        acc2 = acc2 + data[i+1];   /* 仅偶数位元素求前置和 */
    }
    
    /*循环2*/
    for (; i < length; i++) {      /* 累加剩余元素 */
        acc1 = acc1 + data[i];
    }
    *dest = acc1 + acc2;           /* 两个累加器求和 */                            
}
```

代码6中有两个关键路径。

# 9 重新结合变换

考虑对代码5中循环1的计算行：

```C
acc = (acc + data[i]) + data[i+1];
```

做出变换：

```C
acc = acc + (data[i] + data[i+1]);
```

尽管只是对计算式更改了括号的位置，但这对计算性能有了很大的提高。前式的第一次加法`acc + data[i]`前仍然需要等待前一循环acc计算完毕，而后式的第一次加法`data[i] + data[i+1]`则无此要求。利用CPU的*乱序*特性，可以在对于后式计算：可以在计算前一循环acc的同时，去计算后一循环的`data[i] + data[i+1]`，从而提高了效率。

*代码7*

```C
void combine(vec_pointer ptr, data_t *dest)
{
    long int i; 
    long int v_length = length(ptr);         
    long int limit = v_length - 1;      
    data_t *data = get_vec_start(ptr);       
    data_t acc = 0;                          
    
    /*循环1*/
    for (i = 0; i < limit; i+=2) {          
        acc = acc + (data[i] + data[i+1]);   /* 重新结合变换 */
    }
    
    /*循环2*/
    for (; i < length; i++) {                
        acc = acc + data[i];
    }
    *dest = acc;                             
}
```

# 10 一些问题

1. 循环的并行度不能无限提高。一旦平行度超过了可用的寄存器数量，编译器就会把多余的变量存入栈内——从而性能巨减。
2. 现代CPU都有预测分支并提前执行的能力。但是，一旦CPU预测错误，就会造成巨大的性能损失。
    
    * 首先，不要过多的关注可预测的分支(P361);
    * 其次，对于难以预测的情况，尽可能使用**条件数据传送**，而不是*条件控制转移*。
    
> `v = test-expr ? then-expr : else-expr;`的汇编代码可能产生下面两种结果：

```Assembly
/* 条件数据传送 */
vt = then-expr;
v = else-expr;
t = test-expr;
if (t) v = vt;
        
/* 条件控制转移 */
	  if (!test-expr)
		  goto false;
	  v = then-expr;
	  goto done;
  false:
	  v = else-expr;
done:
``` 