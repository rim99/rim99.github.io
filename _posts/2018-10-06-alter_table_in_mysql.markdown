---
layout: post
title: MySQL的Alter table学习笔记
date: 2018-10-06 19:53:02
categories: 笔记
---
 

以前的项目里，数据表的量都很小，从来没有关注过`Alter table`改表的高级用法。最近有点时间，看了下官方文档，发现还是有不少技巧的，记录一下。

# 整理索引碎片

`Alter table`可以修改数据表的数据库引擎。

```SQL
ALTER TABLE table_name ENGINE = InnoDB； 
```

上面这句SQL把数据表的引擎设置成了InnoDB：

* 如果这张表之前不是InnoDB引擎，MySQL会使用新的引擎重建数据表。
* **如果之前就是InnoDB，MySQL也同样会执行操作，将数据表空间重新整理，清理索引碎片。**

InnoDB使用聚簇索引机制存储数据。基于二级索引的随机插入或删除数据会导致索引碎片——也就是，数据行在磁盘的分布顺序与聚簇索引（Cluster Index）顺序并不一致，或者，依据二级索引查询到的数据页块（the 64-page blocks that were allocated to the index）存在很多未使用的数据页。

>数据页（page）是指InnoDB能够单次在磁盘（data files）与内存（buffer pool）之间传输的数据量。同一个MySQL实例的[页大小](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_page_size)是固定的。

索引碎片会造成：

1. 数据表占用过大的空间
2. 查询数据时，如果发生全表扫描，会占用过长时间

> 另外一种解决索引碎片的办法是，使用mysqldump将数据表导出为文本文件，然后重新加载到数据库里。

如果新增数据总是依聚簇索引升序插入的，或者删除的总是最后面的数据行，InnoDB的算法可以保证不会出现索引碎片。

> 聚簇索引
> 
> 1. 如果表定义了主键，InnoDB会优先选用其为聚簇索引
> 
> 2. 如果没有主键，InnoDB会选择第一个UNIQUE索引
> 
> 3. 如果都没有，InnoDB会在一个数据表内部的`row ID`列上建一个隐藏的索引`GEN_CLUST_INDEX`。`row ID`是一个6字节大小的自增数据。

# 性能与空间

`Alter table`语句支持`ALGORITHM`从句选择改表算法。MySQL8.0支持三种算法：

1. INSTANT：这个算法自8.0.12版本引入，只会修改[data dictionary](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_data_dictionary)里的metadata。并发支持度最好。
2. INPLACE：不会复制数据，但会适当的重建数据表，而且会在预备和执行阶段在数据表上设置metadata排他锁。支持并发的执行DML语句。
3. COPY：将旧表数据复制到了新表。不支持并发的执行DML（Data Manipulation Language），但支持并发查询。


`ALGORITHM`从句是可选的，如果没有。MySQL会按INSTANT、INPLACE、COPY依次选择合适的算法。


## INSTANT算法

支持：

- 增加列
- 增加或删除虚拟列
- 增加或删除列的默认值
- 修改索引类型
- 修改ENUM或SET的定义，与INPLACE算法的限制类似
- 表的重命名，与INPLACE算法的限制类似


## INPLACE算法

INPLACE算法支持如下改表操作：

- 对表名、列名、索引名的修改。表名修改后，原先的权限策略不会随之迁移，需要重新授权。
- InnoDB数据表增加或删除二级索引。
- 修改索引可见性。
- 修改列的默认值。
- 删除spatial列的SRID属性。
- 修改列属性时，不会影响生成列（generated column）的值。
- 对列定义的ENUM或SET尾端增加新的字段。

	但是如果：
	- 需要在中间增加字段导致需要对字段重新编号
	- 总字段数量的增加导致占用空间增减，例如原来8个字段增加了一个导致定义的空间从1字节变成了2字节
	
	都需要全表复制。


## COPY算法

COPY算法总是阻塞对数据表的修改操作，而在大部分时候，支持并发查询。但在**清理旧表结构和旧表定义的缓存**的时候，会持有排他锁，不允许查询和写入。

COPY算法会执行数据复制和新建索引，需要富裕的空间才能完成。

如果用于表分区的多列索引的列顺序发生变动，必须使用COPY算法。

# 修改列

有三个关键字可以修改列：

* CHANGE

	- 重命名、修改列定义
	- 比MODIFY功能多一些。即使不需要修改列名，也需要重复声明列名
	- 配合AFTER和BEFORE可以对列重排序
	
* MODIFY

	- 可以修改列定义，不能重命名
	- 配合AFTER和BEFORE可以对列重排序
	
* ALTER

	- 只能修改列的默认值

如果使用CHANGE重命名列，MySQL可以：

* 更新索引的指向
* 更新外键的指向

但不能够：

* 更新生成列、分布表达式的指向
* 视图和存储过程的指向


# 其他

关于生成列和分区表的`Alter table`语句行为，需要再看文档。


# 参考资料

* [15.11.4 Defragmenting a Table](https://dev.mysql.com/doc/refman/8.0/en/innodb-file-defragmenting.html)
* [《MySQL 8.0 Reference Manual  /  ...  /  ALTER TABLE Syntax》](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html)



