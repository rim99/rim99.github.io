---
layout: post
title: "Linux的巨页与透明巨页"
date: 2023-10-28 21:05:29 +0800
categories: 原创
---

Linux的内存管理模块里有两个概念：巨页（hugepage）和透明巨页（transparent hugepage, THP）。

## 内存页面与TLB

Linux的内存管理是以页面（page）为基本单位进行管理的。每一个进程的内存都将内存页面，通过虚拟地址空间的方式映射到物理内存。每一块内存页面内部都对应着连续的物理内存区域。

TLB是CPU中的物理单元，能够缓存内存页面的映射记录。TLB本身的性能很快，接近于寄存器，而高于L1级CPU缓存。但是TLB缓存容量很小。以AMD Zen+架构为例：

缓存类型   | L0  | L1   | L2
:-:       | :-: | :-:  | :-:
TLB指令缓存 | 8条 | 64条 | 512条（不允许1G页面）
TLB数据缓存 | -   | 64条 | 1532条（不允许1G页面）

Linux中的经典内存页面大小为4KB。假如TLB缓存的全部页面是4KB的话，也只能对应8720KB。对于超出范围的内存映射记录，不得不在TLB cache miss之后，另行加载。

8720KB内存，对于现如今的计算生态来说，是远远不够的。

## 巨页

为了提高TLB缓存效率，操作系统引入了巨页这个概念，允许内存页面大小增加到更大：
- X86-64架构支持2MB和1GB两种大小
- ARM支持更多：64kB，2MB，32MB以及1GB

从而，进程内存需要更少的页面映射记录。

### 巨页的缺点

操作系统上可以运行很多进程，不同的进程创建退出，会导致整个系统内存的碎片化。假如某个进程在申请巨页内存的时候，系统恰好在此刻无法找到连续的物理内存区域来满足需求，那就会发生内存分配错误。这种运行时异常，应用程序很可能并没有刻意处理。此时报错会影响到用户体验。

## 透明巨页

为了解决巨页的问题，Linux又引入了透明巨页的概念。在巨页无法正常分配的情况下，系统可以使用透明巨页，继续完成内存分配。而这个过程，用户进程无法感知。所以叫透明。

当系统没有足够大的连续物理内存区域实现内存分配的时候，透明巨页可以使用若干小内存区域完成分配。当稍后有足够的物理内存区域时，系统会自动将进程内存迁移到该区域中，通过khugepaged进程。

目前，Linux支持在 **匿名内存映射** 和 **tmpfs/shmem** 两个场景中使用透明巨页。

### 透明巨页的缺点

可以看出，透明巨页自身也有些缺点：
- 内存页面的物理区域不连续
- 内存还有可能被系统内核自动复制

对于这些，用户进程无从感知。

## 内存管理方式的选择

巨页与透明巨页，两个内存管理模块的名字很相似，然而运行时行为却大相径庭。因此我们需要根据Linux系统运行进程的特征，来选择不同的管理方式。

例如，在桌面环境中，实际运行的进程多种多样，而且不停会有进程创建和退出。这会导致系统运行一段时间后，内存碎片化变得严重起来。为了更好地满足桌面用户需要，透明巨页十分必要。

再比如在服务器场景中，系统运行的进程比较稳定。内存碎片并不严重。那么透明巨页的优势难以发挥，反而其缺点被放大了不少。服务器进程，往往需要很大的内存，非连续内存区域的性能此时较差。而且，内存自动化复制还会对进程的运行性能造成一定的恶化。
在MongoDB和Oracle的官方文档中，透明巨页均被要求关闭。更有甚者，MySQL的非官方引擎TokuDB在透明巨页开启时，会拒绝启动。

Percona有一篇博客[7]()测试了，PostgreSQL 10在透明巨页开启和关闭场景下的性能。结果发现：在透明巨页开启时，数据库的TLB缓存命中率会下降一两个百分点，同时TPS服务能力也存在清晰的差距。

只要进程需要的内存足够大，标准巨页就是有必要的。因为TLB缓存命中率是进程运行性能的保证之一。

## 关于使用

### 巨页

Linux的巨页可用数量，在内核参数`vm.nr_hugepages`中设定。

查看：

```
$> sudo sysctl -a | grep vm.nr_hugepages
vm.nr_hugepages = 0
...
```

在`/etc/sysctl.conf`中修改巨页可用数量：

```
vm.nr_hugepages = 100
```

然后执行命令，使参数生效

```
sysctl -p
```

### 透明巨页

透明巨页开启状态，可以通过文件`/sys/kernel/mm/transparent_hugepage/enabled`来查看。这个文件也是可以修改的，但是系统重启后，设置会被重置。如果想要在启动时设定，需要在`grub.conf`文件中，追加内核启动参数`transparent_hugepage=[never|always]`。

### 监控

如果想要查看给定进程的透明巨页使用情况，可以查看对应进程的`/proc/<pid>/smaps`文件。该文件包含了进程所有的虚拟内存映射项。每一项中都有：
- `AnonHugePages`项，对应于透明巨页所提供的内存空间大小。
- `ShmemPmdMapped`项，对应于标准巨页在进程共享内存（shmem/tmpfs）中的大小。
- `Shared_Hugetlb`和`Private_Hugetlb`对应于标准巨页hugetlbfs页面对应的内存空间。由于历史原因，这些空间没有计算在`RSS`或`PSS`中。

### 其他

更多关于巨页和透明巨页的使用方式，请参考Linux内核文档。

## 参考资料
1. [Transparent Hugepage Support - The Linux Kernel documentation](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)
2. [Zen+ - Microarchitectures - AMD - WikiChip](https://en.wikichip.org/wiki/amd/microarchitectures/zen%2B)
3. [huge_pages - 20.4. Resource Consumption - PostgreSQL Doc](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-HUGE-PAGES)
4. [Disable Transparent Huge Pages (THP) - MongoDB Manual](https://www.mongodb.com/docs/manual/tutorial/transparent-huge-pages/)
5. [Disabling Transparent HugePages - Oracle Database / Release 19](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/disabling-transparent-hugepages.html#)
6. [PostgreSQL and Hugepages: Working with an abundance of memory in modern servers](https://wiki.postgresql.org/images/7/7d/PostgreSQL_and_Huge_pages_-_PGConf.2019.pdf)
7. [Settling the Myth of Transparent HugePages for Databases - Percona](https://www.percona.com/blog/settling-the-myth-of-transparent-hugepages-for-databases/)
8. [How to use, monitor, and disable transparent hugepages in Red Hat Enterprise Linux 6 and 7? ](https://access.redhat.com/solutions/46111)
9. [How to check Transparent HugePage usage per process in Linux with examples](https://www.golinuxcloud.com/check-transparent-hugepage-usage-per-process/)
10. [The /proc Filesystem — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/proc.html)
