---
layout: post
title: HotSpot虚拟机GC调优学习笔记 
date: 2018-06-10 14:49:45
categories: 翻译
---



>本文为[《Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide》](https://docs.oracle.com/javase/10/gctuning/introduction-garbage-collection-tuning.htm#JSGCT-GUID-326EB4CF-8C8C-4267-8355-21AB04F0D304)的学习笔记。该指南为JDK10文档的一部分。

GC（Garbage Collection）指的是在HotSpot虚拟机运行时，自动清理不再使用的内存空间，动态的分配和回收。

# 为什要做GC调优

虚拟机的GC行为是占用计算资源的，会影响到程序运行的效率。而在并行的程序中，这种影响会被放大很多倍。

如果一个算法在单核环境下，只有1%的时间用于GC。当它迁移到32核环境下时，就会损失超过20%的吞吐量。如果在单线程环境的GC用时为10%，在32核环境下就是75%！！！

参见[原图](https://docs.oracle.com/javase/10/gctuning/img/jsgct_dt_005_gph_pc_vs_tp.png)。

因此，降低GC行为的开销是很重要的。

# 调优的原则

指南定义了服务器级计算资源应该满足：

1. 至少有两个CPU核心
2. 至少有2GB的内存

在该条件下，HotSpot虚拟机默认会：

1. 使用G1收集器
2. 初始堆大小为物理内存的1/64
3. 最大堆大小为物理内存的1/4
4. 使用分层编译模式(Tiered compiler)

## 调优指标

GC通常有三个衡量指标：

1. 最大暂停时间。最大暂停时间是指收集器暂停程序运行的时间。这个指标用于控制那些非常耗时的GC行为。主要处于缩短程序响应时间来考虑的。GC收集器会统计每次GC耗时，给近期的GC行为赋予更高的权重，来计算平均耗时。如果平均耗时与统计偏差的和超过了设定目标，收集器会认为该目标没有达成。最大暂停时间目标使用`-XX:MaxGCPauseMillis=<nnn>`来设定。需要注意的是，设定这一目标，容易引发系统GC的行为愈加频繁，从而降低吞吐量。
2. 吞吐量。吞吐量基于GC耗时和程序运行耗时的比例来衡量。该目标使用`-XX:GCTimeRatio=<nnn>`来设定。该参数意味着，程序整体运行时间中，GC耗时不能超过1/(1+nnn)。如果该指标没有得到满足，HotSpot有可能会增加堆区内存大小，以提高程序运行时间。
3. 内存占用量。HotSpot的GC收集器能够在上述目标中，按用户要求，尽力满足其一。在该目标满足的情况下，GC收集器会转而寻求尽力满足另一指标。当两个目标同时得到满足的时候，HotSpot虚拟机会降低内存占用，直到其中一个指标（一般都是吞吐量指标）无法满足。

## 调优的思路

HotSpot在默认情况下，能够动态的调整堆区大小，来满足指定的吞吐量指标。指南建议避免设置过大的Xmx参数（最大堆大小），而应该设置一个满足需要的吞吐量指标。

此外，程序的行为，也会引起堆区空间的扩张与缩小。例如，当一个程序在启动时，申请了一块较大的内存区。随着程序运行，HotSpot有可能会不断扩容以维持程序的吞吐量表现。

如果堆区内存达到了最大值时，吞吐量指标还达不到。这说明，内存分配的不够。建议调整`-Xmx`参数到接近物理内存大小，但不要引起内存swap操作。如果吞吐量还是不够，那说明计算机的硬件资源不足以支持该目标。

如果吞吐量满足要求了，但是暂停时间过高了。那就需要设置最大暂停时间了。需要注意的是，设定该指标后，应用程序有可能无法满足吞吐量指标。这时就需要折衷设置了。

即是程序稳定的时候，HotSpot仍然会不断的收缩扩张堆区，以平衡三个指标的表现。因为，追求吞吐量指标会倾向于增加内存空间，而追求其余两个指标则会倾向于减少内存空间。

# GC收集器的行为简介

HotSpot使用可达性分析来判断一个对象是否可以回收。

大部分对象只会存活很短的时间，而很少的对象有会存活很长时间。基于这种现象，HotSpot的GC收集器采用分代收集的思路，对年轻对象使用“标记-复制”算法，对老年对象使用“标记-清理”或“标记-整理”算法。

HotSpot将内存空间划分为年轻代、老年代，两个大的部分。其中年轻代又进一步细分为Eden区和两个Survivor区。

通常来讲，GC是这样运行的。

新的对象首先出现在Eden区。当Eden区满了之后，就会触发一次Minor GC，将Eden区和在用的Survivor区中的存活对象，复制到另一个Survivor区中。当对象被复制多次后，依然存活，那就会复制到老年代。当老年代满了以后，就会触发一次Major GC，清理老年代中的空间。通常，Major GC的耗时巨大，并会涉及很多对象。

不同用途的应用程序会有不同的性能取向。比如web服务器对吞吐量(Throughput)更看重，但是用户直接操作的UI系统则会对响应延迟（Latency）有要求。此外，占用过多的内存，会影响到应用程序的伸缩性。而对于远程过程调用系统，GC的敏捷性（Promptness），也就是对象从失效到被回收的时间间隔，更为重要。

基本上，调整分代内存空间比例就是这三个指标的相互妥协。例如，较大的年轻代，可以提高吞吐量，但会提高响应延迟、增加内存大小、降低GC敏捷性。

所以说，年轻代和老年代的比例也是内存调优的很重要的一个方面。

使用`-Xlog:gc`或`-Xlog:gc*`来输出GC日志，查看历次GC的行为统计信息。后者输出内容的可读性更佳。

# GC性能影响因素

对GC效率影响最大的两个因素是最大可用内存和年轻代的内存空间比例。

## 最大可用内存

最大可用内存，与应用程序的吞吐量是反相关关系。

当HotSpot的`-Xmx`和`-Xms`参数不相等时，HotSpot会自行调节HotSpot区大小。`-XX:MinHeapFreeRatio`和`-XX:MaxHeapFreeRatio`可以分别设置HotSpot堆闲置区的上下界比例。这两个参数在Solaris系统下默认值分别是40和70。也就是说，当应用程序申请的内存空间增加时，HotSpot会尽力维持40%的闲置区比例，直到HotSpot堆大小达到`-Xmx`值，然后不再增加；当应用程序申请的内存空间减少时，HotSpot会尽力维持70%的闲置区比例，直到HotSpot堆大小达到`-Xms`值，然后不再减少。

指南建议：

1. 尽可能多的给HotSpot分配内存。默认的通常有点小。
2. 最好将HotSpot的`-Xmx`和`-Xms`参数设置为同一个值。减少HotSpot的决策压力。而且，动态地收集释放内存也是不小的开销。
3. 随着系统CPU核数的增加，最好相应的提高内存大小。GC可以并发运行，不会造成开销的增加。

## 年轻代比例

年轻代在HotSpot堆中的比例也是很重要的影响因素。

年轻代比例越大，minor GC的频率就越低；但是，老年代的比例却减少了，major GC变频繁了。

年轻代的大小可以用`-XX:NewRatio`参数来控制。例如，`-XX:NewRatio=3`就意味着年轻代占HotSpot对总大小的1/4。

此外，年轻代的大小还可以用`-XX:NewSize`和`-XX:MaxNewSize`可以控制年轻代内存空间的上下界，实现更细粒度的调优。这两个参数可以类比`-Xms`和`-Xmx`的作用。如果没指定`-XX:MaxNewSize`的值，HotSpot会利用参数`-XX:NewSize`和`-Xmx`来计算这个上限。

使用`-XX:SurvivorRatio`还可以指定Survivor区在年轻代中的比例。Server级HotSpot中的默认参数为8。也就是说，1/10的区域为Survivor区（2个），8/10的区域为Eden区。不过指定，这个参数的调优效果可能并不明显。

如果Survivor区太小，minor GC有可能会将Eden区的对象直接复制到老年代里。如果Survivor太大，就会白费空间。

Survivor区对象复制几次，如果依然存活就会进入老年代（这个过程叫对象老化）。`-XX:MaxTenuringThreshold`可以设置对象的最大复制次数，默认值为15。需要注意的是，这个参数定义的只是一个阈值，并不是实际执行的值。HotSpot运行时，会动态的根据当时内存状况，来复制对象。但并不会严格按照15次的标准将对象移动到老年代。

通过调整老化阈值，将对象尽可能留在年轻代区域，可以提高Survivor区空间的利用率。

指南建议：

1. 先决定好堆的大小，堆大小应该小于物理内存大小，避免出现页错误(page faults)和内存颠簸(thrashing)。
2. 然后增加年轻代空间。需要注意的是，老年代应该在任何时候保持足够大，能够存储所有活跃对象，并保持10%-20%的余地(slack space)。
3. 随着系统CPU核数的增加，最好相应的提高年轻代大小。GC可以并发运行，不会造成开销的增加。

# GC收集器

HotSpot提供四种收集器。

## Serial收集器

使用`-XX:+UseSerialGC`来启用该收集器。

该收集器使用单线程完成所有工作。所以适用场景有限：

1. 单核CPU环境；
2. 多核CPU环境，但是JVM堆大小不超过100MB。

## Parallel收集器

使用`-XX:+UseParallelGC`来启用该收集器。

相比Serial收集器，Parallel收集器可以使用多线程完成GC。如果不启用并行压缩特性(parallel compaction)，major GC会单线程执行。该特性会默认开启，也可以用`-XX:-UseParallelOldGC`来手动关闭。

### 调整GC工作线程数

默认情况下，当HotSpot可用核心数N大于8时，GC工作线程数大约为N * 5/8，否则GC工作线程数大约为N。如果为后者，Parallel收集器工作时，应用程序会完全停滞，失去响应。此外，还可以使用`-XX:ParallelGCThreads=<N>`指定GC工作线程数。

在单核场景下，Parallel收集器会因为线程切换开销造成能效下降，不如Serial收集器。如果可用核心数N大于2，Parallel收集器的效率明显会好很多。

Parallel收集器在minor GC过程中，每一条工作线程在老年代都会申请一块内存区，作为年轻代复制到老年代的“缓冲区”。多次minor GC后，老年代中的内存区域又会出现大量碎片，连片区域减少。如果老年代中的连续区域无法满足某一次的空间申请要求时，就会触发一次major GC，对老年代空间进行压缩。这就是 “碎片效应”。

减少GC工作线程数、增加老年代空间都可以减缓“碎片效应”。

### 分代大小的设定

每一次GC结束后，收集器都会更新统计数据，然后相应的调整分代大小。在这过程中有可能会忽视显式的GC触发操作，例如`System.gc()`。

默认的，内存区每次增加20%，每次减少5%。`-XX:YoungGenerationSizeIncrement=<Y>`和`-XX:TenuredGenerationSizeIncrement=<T>`可以分别用来调整年轻代和老年代的增量幅度。`-XX:AdaptiveSizeDecrementScaleFactor=<D>`可以设定收缩幅度。如果相应代的增幅为X%，其收缩幅度就是X/D%。

在程序启动时，如果遇到内存扩容，收集器会适当增大增幅，以保证更好的性能。收缩不会增加幅度。

如果最大暂停时间指标没有满足，收集器每次收缩一个代的大小，其中暂停时间更大的代优先收缩。

如果吞吐量指标没有满足，年轻代、老年代都会扩容。

### 堆大小的设定

默认参数如“调优的原则”所述。此外，年轻代最大不超过堆总大小的1/3。

如果不确定程序的该用多少内存，可以先试用默认参数，让收集器自行调整。

内存大小的默认值有可能会因为其他参数而修改，可以使用` -XX:+PrintFlagsFinal`参数打印在GC日志里。

## OutOfMemoryError

当收集器用了98%的时间，或者只回收了不超过2%的堆空间。收集器就会抛出OutOfMemoryError。

使用`-XX:-UseGCOverheadLimit`可以关闭这一行为。


## CMS收集器与G1收集器

CMS收集器和G1收集器都会占用一部分计算资源来减少major GC时的卡顿时间。在CPU核心为N时，收集器会调用K/N颗核心，其中1<=K<=ceiling{4/N}。这样就保证GC和应用程序运行能够同时进行，降低了GC时期的响应时间，但也降低了相应时间段的吞吐率。同时这也意味着，单核系统不能使用这两个收集器。


## CMS收集器

CMS收集器自JDK9起弃用，可以使用`-XX:+UseConcMarkSweepGC`手动启用。

如果应用希望有更短的GC暂停时间，并愿意与GC共享计算资源，那么就适合使用CMS收集器。通常如果应用有一些很大而且存活时间较长的对象，使用CMS收集器会有很不错的效果。

### 为什么GC暂停时间短

CMS收集器在major GC时，使用多条线程并发的标记活跃对象、清楚清除不可达对象，而此时应用程序依然保持运行状态。major GC运行中的时候，依然可以触发minor GC。后者类似于Parallel收集器，会引起应用程序暂时失去响应。

### Concurrent Mode 失效

尽管CMS可以和应用程序同时运行，但是如果在老年代空间申请完毕之前，CMS无法及时清理失效对象时，或者说老年代空间无法满足应用程序的要求时，这是就构成一次Concurrent Mode 失效，收集器会暂停应用程序的运行，启动一次major GC。在并发收集过程中，如果遇到一次显式的GC调用`System.gc()`或者诊断工具要求调用GC统计信息时，收集器会上报一个concurrent mode interruption。

### OutOfMemoryError

和Parallel收集器一样，CMS收集器也会在相同的情况下报告OutOfMemoryError。

### 浮动垃圾 

当CMS标记不可对象过程中，一些对象同时被应用程序放弃，但没有被及时的标记到，因此不会在接下来的并发清理过程被清除。成为了浮动垃圾（floating garbage）。浮动垃圾会在下一次major GC循环中被清除。但是，在两次major GC中，如果浮动垃圾过多，会引发Concurrent Mode 失效，进而导致一次stop-the-world全量GC。

指南建议将老年代增大20%，以减缓浮动垃圾的影响。

### CMS收集器的运行

CMS收集器的一轮GC循环包含如下阶段：

1. 初始标记阶段，CMS会收集所有root节点的子一级节点，例如线程栈、寄存器等等。
2. 并发标记阶段，CMS会使用多条线程，标记对象。
3. 重复标记阶段，CMS会查找并发标记阶段没有标记到的对象。
4. 并发清理阶段，CMS会使用多条线程，清理被标记为不可达的对象。

初始标记和重复标记两个阶段的耗时非常短，但是CMS会暂停应用程序的执行。

###  如何触发一次CMS的GC循环

Serial收集器或者Parallel收集器，会在老年代空间满了以后触发一次major GC。而CMS收集器需要保证老年代始终有足够的空间满足应用程序的需要，因此CMS的major GC都是请提前出发的。

CMS收集会收集以往的运行信息来预计老年代空间耗尽的剩余时间和一次GC循环的耗时，然后和尽量在合理的条件下触发GC。但收集器的触发会稍微早一些，因为Concurrent Mode 失效的代价太高了。

另外，CMS收集器也会在老年代空间被消耗一定的百分比后自动触发一次GC循环。`-XX:CMSInitiatingOccupancyFraction=<N>`意味着老年代空间被占用N%后，CMS触发一次GC循环。默认情况下，N为92。

### 暂停的调度

年轻代和老年代的GC是独立进行的，两者不会覆盖同一个时间段，但有可能是连续出现的。如果出现后者情况，应用程序就会感知到一次较为明显的暂停。

CMS收集器，会将重复标记阶段安排在两次年轻代GC暂停之间，来提高应用程序的响应速度。但对于初始标记阶段，没有这种调度。因为后者比前者快多了。


## G1收集器

HotSpot会根据硬件资源，自动启用G1收集器。也可以使用`-XX:+UseG1GC`指定启用。

G1收集器，以CMS收集器替代者的姿态出现，尽可能的平衡低延迟和高吞吐，可以很好地适应如下场景：

1. 堆大小达到了10GB或更多，同时又超过50%的活跃对象；
2. 空间申请和对象老化的数量很大；
3. 堆中有大量的碎片；
4. 预计暂停时间不超过数百毫秒。

### 堆区的划分

不像之前的收集器，G1收集器将整个堆划分为很多个小的连续的区域（region）。一个region是一个基本的内存单元，可以指定给老年代、也可以给新生代，可以是Eden区，也可以是Survior区，当然也可以是空的。而且这个指派在运行时是动态变化的。所以，G1收集器里，年轻代未必是连续的内存区域。而老年代有可能是连续的，以存放大体积对象。

新的对象，一般都首先放在Eden区。如果体积很大，那就会直接放在老年代。

### GC循环

当老年代的占用比例达到某个确定阈值（Initiating Heap Occupancy threshold）的时候，G1收集器触发一次GC循环。

1. Young-only阶段
    1. 初始标记。并发的标记出老年代里的所有可达对象，用于后面的Space-reclamation阶段。在此过程中，也可以触发一次年轻代的GC。接下来是两个stop-the-world暂停。
    2. 再标记。结束初始标记阶段，处理全局引用和类卸载。然后并发的计算出活跃对象的整体信息。该信息将交给清理阶段。
    3. 清理。根据上一阶段的全局对象统计信息，更新内部的数据布局。回收多余空间，然后决定是否执行进入Space-reclamation阶段。如果不进入，就会执行一次Young-only的清理工作。
2. Space-reclamation阶段。这个阶段实际上是年轻代和老年代的混合清理。当G1收集器认为继续清理不划算的时候，该阶段就会结束。

Space-reclamation阶段之后，G1收集器会再触发一次Young-only GC。

如果G1收集器在统计活跃信息就没有足够的内存空间时，G1就会触发一次stop-the-world全量GC。该全量GC是一个并发的“标记-扫描-压缩”过程。效果与Parallel收集器的老年代GC类似。

### G1收集器的细节

#### 初始堆占比

初始堆占比（IHOP，Initiating Heap Occupancy Percent），是Young-only的初始标记阶段的触发阈值，是老年代的占用比例。

G1收集器默认会开启适应式IHOP特性。在这种情况下，G1会观察对象标记过程的耗时和老年代内存收集的效率，来动态的调整`-XX:InitiatingHeapOccupancyPercent`，也就是IHOP阈值。可以使用`-XX:-G1UseAdaptiveIHOP`来关闭这一特性，这样G1就会严格的在满足给定IHOP阈值时触发GC。

适应式IHOP特性，实际上，将第一次触发Space-reclamation阶段时的老年代空间作为一个峰值max，`-XX:G1HeapReservePercent=<buffer_size>`参数作为一个冗余缓冲空间。IHOP = max - buffer_size。

#### 标记

G1使用一种SATB（Snapshot-At-The-Beginning）算法，在标记的初始阶段，对整个堆拍取快照，将其中的活跃对象纳入后续的标记过程中。这个算法的好处是，暂停时间很短；缺点是容易导致后边清理过程中保留不必要的对象。这些被保留的对象会在下次GC过程中被处理。

#### 堆区空间不足

当应用程序保留了很多内存，导致收集器找不到足够的空间完成复制清理工作的时候，收集器会发出evacuation failure。当发出该错误后，G1会停止当前阶段，继续执行后续阶段。G1收集器，通常假设该错误出现在GC的后期，也就是：大多数对象完成移动，剩余空间足以支撑应用程序运行直到Space-reclamation阶段启动。但如果，该假设失败了，G1收集器就会七佛那过一次全量GC。

#### 超大对象（Humongous Objects）

默认的，超大对象是指体积超过一个region的50%的对象。可以使用参数`-XX:G1HeapRegionSize`指定该比例。

这些超大对象通常用特殊的方式来对待：

1. 每一个超大对象都存储在一段连续的region片中。对象的存储起点就是第一个region的起点，而最后一个region的剩余空间都无法再利用。如果有太多这样的对象，就会出现空间浪费，并频繁引发全量GC。
2. 通常，超大对象只会在清理阶段，或者在全量GC阶段中被回收。但如果这个对象是一个基本类型的数组，例如`int[]`，G1收集器会尝试在这个对象在任何GC阶段没有被其他对象引用时，尝试回收。这一行为默认开启，可以使用`-XX:G1EagerReclaimHumongousObjects`来关闭。
3. 回收超大对象通常会导致很长时间的暂停。G1收集器会优先检查超大对象占用是否超过了IHOP，如果是，则立即停止当前的标记过程，直接处理超大对象的清理工作。

#### Young-Only阶段

在Young-Only阶段，G1收集有年轻代的内存占用情况。在Young-Only回收阶段，G1收集器会根据年轻代收集的历史数据（对象的复制耗时以及相互引用的复杂程度）来评估年轻代各个region，按照暂停时间指标来设定指标`-XX:MaxGCPauseTimeMillis`和`-XX:PauseTimeIntervalMillis`，从而控制GC耗时。

如果没有其他约束，G1收集器会根据`-XX:G1NewSizePercent`和`-XX:G1MaxNewSizePercent`两项指标决定年轻代的大小以满足最大暂定时间指标。

#### Space-Reclamation阶段

在Space-Reclamation阶段，G1会尽可能在单次GC暂停中回收尽可能多的老年代内存。年轻代的回收目标一般计划为`-XX:G1NewSizePercent`。而老年代region会增加直到G1收集器认为最大暂停时间难以维持。在一次GC暂停中，G1收集根据回收效率决定回收次序，效率高的优先。

region的占用比例低于`-XX:G1MixedGCLiveThresholdPercent`时，会成为回收候选region。G1收集器会将所有候选回收region列入一个集合。每一次GC暂停只会回收有限个数的region，用`-XX:G1MixedGCCountTarget`来指定。

当集合中所有可回收空间占比低于`-XX:G1HeapWastePercent`时，Space-Reclamation阶段阶段结束。

#### G1收集器的调优策略

G1收集器可以很好的平衡吞吐率和GC延迟两项指标。如果对吞吐率有要求，应该指定一个较大的`-XX:MaxGCPauseMillis`参数或者提供更大的堆空间。如果对延迟有要求，应该制定一个小一点的`-XX:MaxGCPauseMillis`参数。尽量不要使用`-Xmn`、`-XX:NewRatio`这样的参数限制年轻代的大小，因为G1收集器主要依赖年轻代大小来控制GC延迟。

如果应用程序从其他收集器迁移过来，只需要设定GC延迟时间和堆的大小就好了。其他参数都没什么用，甚至还会有反作用。

使用`-Xlog:gc*=debug`参数收集G1的GC信息。

##### 观察全量GC

全量GC非常耗时，应该尽量避免。

在日志中查找`Pause Full (Allocation Failure)`关键字可以定位到全量GC。通常，全量GC是evacuation failure引起的，该错误的关键字是`to-space exhausted`。

通常减少全量GC的策略有：

1. 如果是因为大量的超大对象，`gc+heap=info`日志可以展示出超大对象区的数值。每次GC后，最好降低对象数量。可以通过`-XX:G1HeapRegionSize`调大region的空间大小，来降低超大对象区的大小。当前region大小在日志的开始有打印。需要注意的是，增大`-XX:G1HeapRegionSize`可能会出现大量的内存空间因位于超大对象区的最后一个region而被浪费。
2. 增加堆大小。通常会引起标记时间的增加。
3. 使用`-XX:ConcGCThreads`参数增加GC工作线程数。
4. 让G1提前启动GC。
    1. 调高`-XX:G1ReservePercent`增加缓冲区大小，从而降低触发Space-Reclamation阶段的阈值IHOP；
    2. 使用`-XX:-G1UseAdaptiveIHOP`和`-XX:InitiatingHeapOccupancyPercent`关闭适应式IHOP特性，并赋予一个更低的触发阈值。

对于`System.gc()`这样的显式调用，可以使用`-XX:+ExplicitGCInvokesConcurrent`来缓解，或者用`-XX:+DisableExplicitGC`干脆忽略。

##### 如果GC延迟过长

**不寻常的User时间或Real时间**

使用`gc+cpu=info`参数输出的日志里包含了每次暂停的耗时，例如`User=0.19s Sys=0.00s Real=0.01s`这样。其中User时间花费在虚拟机指令上，System时间是操作系统造成的，Real时间才是真正的GC停顿。

System时间过高的话，一般都是环境造成的。

1. 有可能是虚拟机向操作系统申请或归还内存造成的耗时。可以用`-Xms`和`-Xmx`将堆大小设置为定值，然后用`-XX:+AlwaysPreTouch`在虚拟机启动时获取所有堆内存；
2. Linux系统中，Transparent Huge Pages（THP）特性会将小的内存页合并为大的内存页，以减少系统内存查询开销。内存页合并的时候会有很大的开销，而JVM不需要利用这一项功能。建议参照系统手册将其关闭。（话说这货还挺恼人，MongoDB也要求关掉）
3. 日志输出导致磁盘IO占用，也有可能提高GC延迟时间。建议使用远程输出，或者输出到不同的磁盘上。

如果Real时间过大，这个往往说明GC没有足够的CPU运行时间，也就是说机器过载严重。

**引用对象处理耗时过长**

在`Ref Proc`阶段，G1收集器根据对象类型的要求更新引用对象的引用（updates the referents of Reference Objects）。在`Ref Enq`阶段，G1收集器将引用方失效的对象纳入各自的引用队列。

如果这两个阶段耗时过长，应该考虑使用`-XX:+ParallelRefProcEnabled`参数并行化这两个过程。

**Young-Only清理时间过长**

年轻代的清理时间，与年轻代的空间大小，或者说年轻代中需要被复制的对象大小呈正相关关系。

如果是`Object Copy`阶段下的`Evacuate Collection Set`子阶段耗时过长，应该减少`-XX:G1NewSizePercent`。

如果GC后的存活对象大多，会导致GC暂停时间出现峰值。减少`-XX:G1MaxNewSizePercent`降低年轻代大小，也会降低单次GC的复制对象数量。

**混合清理时间过长**

使用`gc+ergo+cset=trace`查看GC日志中的输出，通过`predicted young region time`和`predicted old region time`分别查看年轻代和老年代的回收时间。

如果是老年代暂停时间过长：

1. 增加`-XX:G1MixedGCCountTarget`参数，将老年代的空间回收分布在多次GC循环中；
2. 高占有率的region需要花很多时间来回收。可以调整`-XX:G1MixedGCLiveThresholdPercent`参数，占用比率高于该参数的region不会被放入候选集合。
3. 增加`-XX:G1HeapWastePercent`可以让G1收集器提前放弃回收更多老年代空间。

**remembered set的更新、扫描时间过长**

remembered set记录的是跨region对象的关联关系。当对象在region间复制的时候，remembered set也应该需要更新。但是处于效率的考虑，这个更新操作是延迟的，以便批量执行。

而在GC阶段，remembered set记录需要保持最新，因此remembered set记录的更新会花费一些时间。而remembered set记录的扫描过程，会搜索对象引用、移动region内容然后更新remembered set记录里的对象引用关系。这个阶段也比较耗时。

增加`-XX:G1HeapRegionSize`可以减少region大小，也可以减少跨region的引用关系数量。从而减少这个因素引起的GC卡顿时间。

此外，减少`-XX:G1RSetUpdatingPauseTimePercent`会让G1收集器并发的执行remembered set记录的更新工作。

某些优化工作会通过批量执行更新任务，来减少并发更新，如果应用程序出现大体积对象时，而且这刚好在GC之前，GC时候的remembered set记录的更新就会消耗很长的时间。使用`-XX:-ReduceInitialCardMarks`可以关闭该行为。

G1收集还会压缩remembered set的存储大小。压缩率越大，remembered set记录的扫描工作就越耗时。G1收集器压缩remembered set记录的行为叫做`remembered set coarsening`。使用`-XX:G1SummarizeRSetStatsPeriod`和`gc+remset=trace`来观察日志中是否有这一行为。在`Before GC Summary`区域中的`Did <X> coarsenings`航，X就是remembered set的压缩率，越大越耗时。增加`-XX:G1RSetRegionEntries`可以降低压缩率。另外注意在生产环境中避免打印该日志，因为收集该数据也有不小的开销。

##### 如果吞吐率不足

如果应用程序对吞吐率有要求，那就应该减少GC的频率。可以使用`-XX:MaxGCPauseMillis`增大最大暂停时间，提高每次GC的收集内存数量。启发式分代调整策略，会增加年轻代大小，从而降低GC频率。增加`-XX:G1NewSizePercent`也可以强制G1收集器提高年轻代的下限大小。

某些情况下，`-XX:G1MaxNewSizePercent`，年轻代最大的允许大小，会因为限制了年轻代的大小，而限制了吞吐率。可以通过查看`gc+heap=info`日志来确认是否有这种行为，此时Eden区和Survivor区大小的加和与`-XX:G1MaxNewSizePercent`很接近。可以适当提高`-XX:G1MaxNewSizePercent`来避开限制。

emembered set记录的并发更新是非常消耗CPU的任务。降低并发工作量也可以提高吞吐率。可以增加`-XX:G1RSetUpdatingPauseTimePercent`将并发更新的工作量移动到GC阶段。在最坏的情况下，可以使用`-XX:-G1UseAdaptiveConcRefinement -XX:G1ConcRefinementGreenZone=2G -XX:G1ConcRefinementThreads=0`来关闭emembered set记录的并发更新。此时所有的更新都在GC时执行。

使用`-XX:+UseLargePages`开启大页面支持，也有助于吞吐率的提高。可以查看系统手册，开启这一特性。

虚拟机向操作系统申请或归还内存造成的耗时也会降低应用程序的吞吐率，前文对这个已有介绍。

##### 如果堆区过大

使用`-XX:GCTimeRatio`调整。


### 总结一下

1. Parallel收集器将老年代看作一个整体来回收，而G1则分为很多区域来分别处理。因此G1收集器的延迟更低，但吞吐率可能不好。
2. CMS无法及时的处理老年代碎片，只能被动地等待全量GC；G1收集器可以较好的避免这个问题。
3. G1收集器对资源占用偏多，会影响到应用程序的效率。
4. G1收集器可以在minor GC中回收空的大型老年代对象，避免了一些低效率的major GC。默认开启，使用`-XX:-G1EagerReclaimHumongousObjects`关闭。
5. G1收集器可以剔除重名的字符串对象。默认关闭，可以使用`-XX:+G1EnableStringDeduplication`开启。


## 指南建议

优先调整堆大小来调优性能。如果内存调优无法满足要求，在考虑改变默认收集器。

1. 如果程序占用内存不大（不超过100MB），或者只调用一个线程而且没有最大暂停时间限制，那就用Serial收集器；
2. 如果程序的峰值性能或者吞吐量很重要，或者对最大暂停时间没有要求（超过1s），那就可以使用Parallel收集器；
3. 如果对响应时间的要求比较高，例如GC暂停时间不能超过1s，那推荐使用G1收集器或CMS收集器。

GC的性能依赖于堆大小、应用的活跃对象数量以及可用的CPU核心数和CPU性能，所以，上面说的1s并不是绝对的，Parallel收集器的major GC有可能会有很长时间的停顿，G1收集器或CMS收集器的GC暂停时间也很有可能超过1s。

# 延伸阅读

* [MaxTenuringThreshold - how exactly it works?](https://stackoverflow.com/questions/13543468/maxtenuringthreshold-how-exactly-it-works)
* [从实际案例聊聊Java应用的GC优化](https://tech.meituan.com/jvm_optimize.html)
* [JVM 优化经验总结](https://www.ibm.com/developerworks/cn/java/j-lo-jvm-optimize-experience/index.html)
* [JVM调优总结 -Xms -Xmx -Xmn -Xss](http://unixboy.iteye.com/blog/174173)
* [JVM参数（三）打印所有XX参数及值](http://www.cnblogs.com/duanxz/p/6098908.html)
* [G1 - Never Done!](https://fosdem.org/2018/schedule/event/g1/attachments/slides/2341/export/events/attachments/g1/slides/2341/fosdem2018_schatzl_g1_never_done.pdf)




















