---
layout: post
title: 几种数据库的数据存储结构 
date: 2018-12-23 20:00:59
categories: 笔记
---



# 聚簇索引

MySQL的InnoDB引擎是使用聚簇索引的典型代表。

InnoDB是一个以行为单位的数据引擎。

1. 每一行数据依次存储在磁盘上。每一行的主键则联合起来组成一个B+树索引。
2. InnoDB表的主键是唯一获取行数据的key值。也就是说每当要查询某一行数据的时候，InnoDB必须通过主键的B+树来查询该行。所以所有数据表的非主键索引（二级索引）都会在叶子结点上存储主键。
3. InnoDB的新增操作，会尽量按照主键的顺序将数据写在合适的位置，但也有可能不严格执行。如果是后一种情况，会产生数据库索引碎片，影响读取效率。
4. 如果要修改某一行数据，则需要通过索引定位到具体行来完成修改。也有可能新修改的数据太大存不到原来的位置，同样会产生索引碎片。

## 小结一下

如果已经知道主键，那么在InnoDB中读取数据行是非常快的，只需要完成一颗B+树的查询即可。

如果不知道主键，而是通过二级索引来查询，则需要查询两颗B+树。

如果新增数据行的主键总是为主键索引树中的最大值，那么数据行总插入在表文件的最末尾，速度较快；如果新增不满足最大值的要求，那么插入新数据，可能导致InnoDB去磁盘读取一次B+树块，降低写入效率。

# 堆表结构

PostgreSQL的数据库引擎不可更换，而且只支持堆表结构，不支持聚簇索引。

在PostgreSQL中，数据表都至少有两部分：一个是主键的B+树索引，另一个是行数据所在的堆表。主键索引树的叶子结点包含有行数据的指针。

与MySQL的InnoDB不同，PostgreSQL的没有二级索引的说法，非主键索引的叶子结点依然是行数据的指针。

PostgreSQL在新增、修改和删除数据行时，效率很高。因为新的数据都一整行插入到堆表文件的末尾，然后修改各个索引树叶子节点的行指针。之前的行数据被标记为不可用，而没有实时删除。这个“效率高”仅仅指的是与客户端交互的通信时间很短。实际上，随着数据表行数据的反复修改删除，表内无用的数据越来越多。这个现象就是“表膨胀”。为了提高读取效率和磁盘空间利用率，需要对数据表空间进行一次压缩整理。这个操作就是`vacuum`。PostgreSQL在执行`vacuum`操作时，会降低运行效率，实际上是一种“写放大”现象。

如果是全行扫描，PostgreSQL总会按照行的物理分布，顺序读取。而MySQL则会依照主键索引的顺序来读取行，如果行没有严格按顺序写入的话，就会发生磁盘的随机IO，届时性能会出现严重下降。

## 小结一下

无论是否知道主键，只要查询key建立有索引树，那么读某一行的效率是一样的：两次磁盘IO，一次通过索引查询行指针，一次根据指针读取行数据。

对于写操作，PostgreSQL避免了检索行数据之前的位置，只完成磁盘顺序写入和行指针的修改，所以速（tou）度（ji）很（qu）快（qiao）。需要注意的是写放大的问题。可以关注一下PostgreSQL在vacuum方面的进展。


# HBase的LSM索引

LSM索引的产品多了去了，HBase、Cassandra、Google LevelDB以及MongoDB默认的WiredTiger引擎等都是LSM索引的拥趸。不过他们的实现原理大同小异。所以这里以HBase为代表介绍。

每当有新的数据写入HBase的时候，这份数据都会先暂存在内存中。HBase中以列族为单位，划分出不同的内存空间，用MemStore对象来实现。MemStore的核心为一个ConcurrentSkipList。每一份数据插入ConcurrentSkipList后，都会按rowkey的大小来排序。

当MemStore的数据达到一定的阈值的时候，这个数据就会刷新到磁盘成为一个HFile。当磁盘中小体积的HFile足够多的时候，这些文件就又会合并成为一个大的HFile，最终刷入HDFS。

当HBase需要读取某一个rowkey对应的数据时，HBase在检查完BlockCache后，会检查本地的StoreFile和远程的HFile。查询成千上万份小文件当然是很低效的，因此需要有一个合并的过程。

所有持久化到磁盘的HFile文件都是不可改的，所以和PostgreSQL类似，HBase的每一次修改删除都是在数据库中产生新的数据，或者是一个墓碑标记。当HBase执行Major Compaction合并文件的时候，旧的文件记录才会删除。

当然了，每一个HFile都读一下是很低效的，所以HBase引入Bloom过滤器，来判定某个文件中是否**不存在**要查询的rowkey。

[SSTable and Log Structured Storage: LevelDB](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)一文中还介绍了LevelDB用到的`rowkey:offset`式的索引，不过在HBase文档中没看到相关介绍。

## 小结一下

尽管B+树索引和LSM索引都认同磁盘的顺序读写对数据库运行效率的影响，但是他们两者的侧重点不同。

1. B+树聚簇索引为了实现高效读取，不惜限制数据使用有序主键写入数据，但是如果要求主键有序往往会造成主键成为一个多线程同步的热点，降低了写入性能；
2. LSM索引在读取数据的时候是顺序读盘，但是效率偏低，因为有些本来没有数据的文件因为Bloom过滤器没能拦截，导致无效全文读取，不过HBase写入数据时不需要更新二级索引等等，所以写入效率很高。

B+树堆表结构和LSM索引在写入方面都通过追加新数据、墓碑标记等操作实现数据的修改和删除，所以都有写放大的问题。有人尝试过将HBase的所有Compaction操作都推迟到HDFS层执行，发现“写放大”有显著降低。此外，写放大问题也是[Uber抛弃PG重返MySQL](https://eng.uber.com/mysql-migration/)的原因之一。

为了减少写放大，HBase可以通过识别业务特征选择不同的Compaction策略。

还有一种策略来自另一款LSM索引产品LevelDB：通过分层的控制文件规模，来减少文件合并次数。

# 参考资料

* [Which queries are faster with Postgres than with MySQL InnoDB](https://dba.stackovernet.com/cn/q/41443)
* [高性能MySQL](https://read.douban.com/ebook/3564856)
* [一条数据的HBase之旅，简明HBase入门教程-Write全流程](http://www.nosqlnotes.com/technotes/hbase/hbase-overview-writeflow/)
* [一条数据的HBase之旅，简明HBase入门教程-Read全流程](http://www.nosqlnotes.com/technotes/hbase/hbase-read/)
* [一条数据的HBase之旅，简明HBase入门教程-Flush与Compaction](http://www.nosqlnotes.com/technotes/hbase/flush-compaction/)
* [HBase不睡觉书](https://read.douban.com/ebook/51046818/)
* [SSTable and Log Structured Storage: LevelDB](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)
* [Wired Tiger - Log-Structured Merge Trees](http://source.wiredtiger.com/2.3.1/lsm.html)

# 延伸阅读

* [MongoDB Benchmark: Btree vs LSM](https://github.com/wiredtiger/wiredtiger/wiki/Btree-vs-LSM)
* [How PostgreSQL Organizes Data](http://etutorials.org/SQL/Postgresql/Part+I+General+PostgreSQL+Use/Chapter+4.+Performance/How+PostgreSQL+Organizes+Data/)
* [Postgresql block internals](https://fritshoogland.wordpress.com/2017/07/01/postgresql-block-internals/)
* [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
* [Analysis of HDFS Under HBase: A Facebook Messages Case Study](https://research.fb.com/publications/analysis-of-hdfs-under-hbase-a-facebook-messages-case-study/)


