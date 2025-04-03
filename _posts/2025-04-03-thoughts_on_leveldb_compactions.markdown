---
layout: post
title: "对LevelDB Compaction抽象的思考"
date: 2025-04-03 22:37:22 +0800
categories: 原创
---

LevelDB是Google在2011年发布的一个嵌入式单机数据库引擎。

在当时，机械硬盘是服务器的主流存储设备，如何充分利用机械硬盘的IO性能，是业界需要解决的一个问题。

而LevelDB给出的答案是LSM树。

## LSM树

LSM树将键值对数据按照键key的字节序排序，以跳表skiplist存在内存中，被称为MemTable。当MemTable的内容增加到一定大小的时候，数据库写入新的MemTable，并把老的MemTable写入到磁盘中，生成SSTable文件。

按LevelDB默认行为，新生成的SSTable文件，列入L0级。然后每当level0里的的SSTable文件数量达到一定阈值，就会进行文件合并，生成新SSTable文件，并列入L1级。

此后，每当某一段键key的范围分布在多个SSTable文件，甚至分布于多个Level中，这些SSTable文件也会合并，生成新的SSTable文件，并提升一个Level。

总的来看，LevelDB把杂乱写入的键值对，先在内存中做好排序，然后将排序好的大段的数据，一次性写入磁盘。本质上，把数据的随机写入，转换成机械硬盘的顺序写。而在查询时，把按key随机查找转换成，对少量SSTable文件的随机查找，和单个文件内部的有序键值对集合的logN查找。

## Compaction

LevelDB发布之初影响很大，但其内部设计也有一些小瑕疵。例如上述过程中，MemTable写成SSTable文件，和SSTable文件合并，都是同一个后台线程依次完成的。

尽管LevelDB会生成两个MemTable对象轮换写入，但假如MemTable需要写成SSTable文件时，后台线程正在合并很多（或者很大的）SSTable文件，就会发生线程阻塞。如果LevelDB正在高速写入数据，导致两个MemTable都已经写满，都需要写成文件时，LevelDB则立即阻塞写入操作，直到后台线程完成合并。

在LevelDB内部实现里，前述两种行为都被抽象为Compaction：
- MemTable写成SSTable文件，为Minor Compaction
- SSTable文件合并，为Major Compaction

所有Compaction行为，都由同一个后台线程完成。

乍一听，好像挺有道理。因为Major Compaction需要对多个文件读取、合并，对于当时IO能力弱的机械硬盘来说，只保留一个后台线程来工作，更有效率。

但是，Minor Compaction，原本是一个需要尽快完成的操作，却被Major Compaction在设计上的权衡给影响了。

这俩本不应该是一类行为，应该分开设计，但被共同的抽象给局限了。

## 对Compaction抽象的反思

这个Minor Compaction本质上是一个内存刷盘的行为，称为Flush不为过。

出于最简改动，Flush和Major Compaction可以共享同一个线程池。线程池可以是弹性的：如果需要Flush的时候，已有的线程正在执行Major Compaction，就立刻新建线程执行Flush，保证Flush的及时性。线程池，只需保证闲时至少有一个线程能够尽快响应即可。更简单的实现，不如干脆来一个双线程的线程池，把线程回收的逻辑也省去。

## RocksDB的改进

RocksDB在LevelDB的基础上，对Compaction进行了改进。

Minor Compaction，改称Flush。每一个DB（其实是列族Column Families，但这是另一个故事了）都持有一个MemTableList，从而可以有不限于2个的MemTable对象作为后台线程的就绪任务。而Major Compaction，简称为Compaction。

在RocksDB中，Flush和Compaction都用后台线程池管理，但分别由BGJobLimits里不同的槽位来控制并发度。

> Wow，一不小心，点评了一下Jeff Dean的代码，真是Big胆！

