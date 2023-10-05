---
layout: post
title: "Postgres存储引擎性能简测：Hydra vs Heap"
date: 2023-10-05 11:28:23 +0800
categories: 原创
---

Hydra是为Postgres设计的OLAP引擎。和原生的Heap引擎相比，它的优势究竟如何呢？

最近在Aliyun上做了简单的测试对比了一下，感觉名副其实。

## 测试环境

Server端操作系统使用了Fedora官方镜像`Fedora-Cloud-Base-38-1.6.x86_64.qcow2`，自行编译安装了Postgres 14.9。

Client端操作系统选择了Arch官方镜像`Arch-Linux-x86_64-cloudimg-20230915.178838.qcow2`，自行编译安装了Postgres 14.9。

服务器选择了`ecs.r6.xlarge`，为4vCore+32GB。服务器CPU为Intel Xeon Platinum 8269CY（Cascade Lake），2.5GHz主频，睿频3.2GHz。

数据挂载使用了ESSD AutoPL型块存储，大小为100GB，开起了性能突发模式。（血泪的教训，千万不要开启这个模式，哪怕用大一点的硬盘也比这个强）。

## 测试工具

这里直接选用了Postgres官方提供了pgbench工具。pgbench主要针对OLTP场景，因此在OLAP场景下，自己另外选择了几个分析性SQL语句，简单的对比了一下。

pgbench的数据集模拟了银行储户场景。其中，pgbench_branches表存储银行机构，pgbench_accounts表存储储蓄账户，pgbench_history表存储储蓄账户的交易记录。每家银行机构对应10万储蓄账户。如果需要增加数据集大小，可以在初始化的时候提供更大的scale factor因子，工具可以生成更多的银行机构数量。

## 数据集初始化

这次scale factor选择了4000。也就是，生成4000个银行机构，和4亿储蓄账户。

```
/pg/14/bin/pgbench -i -s 4000 -h ${PG_HOST} -U postgres postgres
```

在Heap引擎中，生成的数据大小为61GB。


在Hydra引擎中，生成的数据大小仅为12GB。五分之一！OMG，硬盘空间还是很节省的。

用时对比

Engine     |  Total Time | Insert Time | Vacuum Time | Primary Key Index Creation Time
 :-:       |  :-:        | :-:         |  :-:        | :-:
 Heap      |  1534.9     | 515.27      | 519.48      | 500.15
 Hydra     |  478.46     | 273.79      | 25.36       | 179.30

可以看出，无论是批量插入数据，还是Vacuum、索引构建这种维护性操作，Hydra在用时上都大为节省。

## 单点查询测试

单点查询对于OLTP场景更重要。Hydra因为列式存储的原因，并不擅长此场景。

```
/pg/14/bin/pgbench -c 100 -j 150 -t 1000 -h ${PG_HOST} --select-only -U postgres postgres
```

对Heap引擎，测试结果还不错

```
pgbench (14.9)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 4000
query mode: simple
number of clients: 100
number of threads: 1
number of transactions per client: 100
number of transactions actually processed: 10000/10000
latency average = 21.048 ms
latency stddev = 14.351 ms
initial connection time = 229.074 ms
tps = 4468.516841 (without initial connection time)
```

但对于Hydra引擎，上述参数测试极慢。这里命令会每10s报告一次阶段性的结果，做个参考。

```
$> /pg/14/bin/pgbench -c 100 -j 150 -t 1000 -h ${PG_HOST} --select-only -U postgres -P 10 postgres
pgbench (14.9)
starting vacuum...end.
progress: 10.0 s, 288.5 tps, lat 337.378 ms stddev 34.884
progress: 20.0 s, 302.1 tps, lat 331.063 ms stddev 25.183
progress: 30.0 s, 301.8 tps, lat 331.303 ms stddev 22.239
progress: 40.0 s, 303.1 tps, lat 331.062 ms stddev 23.108
progress: 50.0 s, 299.2 tps, lat 334.344 ms stddev 23.207
progress: 60.0 s, 300.4 tps, lat 332.583 ms stddev 23.476
progress: 70.0 s, 300.3 tps, lat 332.280 ms stddev 24.434
progress: 80.0 s, 301.6 tps, lat 332.331 ms stddev 27.919
^C
```

## 分析性查询

在初始化后的数据集上，这里跑了一下pgbench的默认脚本。默认脚本能够，修改储蓄账户的当前账额，增加交易记录。当储蓄账户有了变更，就可以做一些分析了。

```
/pg/14/bin/pgbench -c 10 -j 15 -t 100 -h ${PG_HOST} -U postgres postgres
```

### 分析场景一

这个场景较为简单。找出变更后的pgbench_accounts表中最大的100个账户。

```sql
select * from pgbench_accounts order by abalance desc limit 100;
```

在Heap引擎中，explain analyze结果是这样的

```
QUERY PLAN                                                                            
----------
 Limit  (cost=14595447.87..14595459.54 rows=100 width=97) (actual time=256246.481..256252.433 rows=100 loops=1)
   ->  Gather Merge  (cost=14595447.87..53488445.92 rows=333345280 width=97) (actual time=256246.480..256252.424 rows=100 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=14594447.85..15011129.45 rows=166672640 width=97) (actual time=256243.351..256243.356 rows=67 loops=3)
               Sort Key: abalance DESC
               Sort Method: top-N heapsort  Memory: 39kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 39kB
               ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..8224339.40 rows=166672640 width=97) (actual time=0.341..246211.910 rows=133333333 loops=3)
 Planning Time: 0.073 ms
 Execution Time: 256252.458 ms
(12 rows)
```

PG启动了2个worker进程，并行地扫描了全表。

而在Hydra引擎中，explain analyze结果是这样的

```
QUERY PLAN                                                                                   
----------
 Limit  (cost=2206605.35..2206617.64 rows=100 width=97) (actual time=40191.632..40198.202 rows=100 loops=1)
   ->  Gather Merge  (cost=2206605.35..51349585.35 rows=400001000 width=97) (actual time=40191.630..40198.193 rows=100 loops=1)
         Workers Planned: 7
         Workers Launched: 7
         ->  Sort  (cost=2205605.23..2348462.73 rows=57143000 width=97) (actual time=40150.003..40150.010 rows=88 loops=8)
               Sort Key: abalance DESC
               Sort Method: top-N heapsort  Memory: 39kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 2:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 3:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 4:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 5:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 6:  Sort Method: top-N heapsort  Memory: 39kB
               ->  Parallel Custom Scan (ColumnarScan) on pgbench_accounts  (cost=0.00..21640.86 rows=57143000 width=97) (actual time=11.977..18766.711 rows=50000000 loops=8)
                     Columnar Projected Columns: aid, bid, abalance, filler
 Planning Time: 4.511 ms
 Execution Time: 40198.252 ms
(18 rows)
```

PG无视了默认参数启动了7个worker进程，并行地扫描了全表。

当修改了默认参数后，`set max_parallel_workers_per_gather = 8`，Heap引擎的表现稍有加强，但仍然劣于Hydra的表现

```
postgres=# explain analyze select * from pgbench_accounts order by abalance desc limit 100;
QUERY PLAN                                                                           
----------
 Limit  (cost=8969663.60..8969675.96 rows=100 width=97) (actual time=127956.821..127957.997 rows=100 loops=1)
   ->  Gather Merge  (cost=8969663.60..58436286.42 rows=400014336 width=97) (actual time=127956.819..127957.986 rows=100 loops=1)
         Workers Planned: 8
         Workers Launched: 7
         ->  Sort  (cost=8968663.46..9093667.94 rows=50001792 width=97) (actual time=127946.697..127946.707 rows=90 loops=8)
               Sort Key: abalance DESC
               Sort Method: top-N heapsort  Memory: 39kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 2:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 3:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 4:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 5:  Sort Method: top-N heapsort  Memory: 39kB
               Worker 6:  Sort Method: top-N heapsort  Memory: 39kB
               ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..7057630.92 rows=50001792 width=97) (actual time=0.118..120697.600 rows=50000000 loops=8)
 Planning Time: 0.066 ms
 Execution Time: 127958.024 ms
(17 rows)
```

只有给abalance列添加索引后，Heap引擎的分析时间才减下来。但这个属于空间换时间了。

```
postgres=# CREATE INDEX acc_abalance_idx ON pgbench_accounts (abalance);
CREATE INDEX
Time: 425301.879 ms (07:05.302)
```

```
postgres=# explain analyze select * from pgbench_accounts order by abalance desc limit 100;
QUERY PLAN                                                                             
----------
 Limit  (cost=0.57..4.05 rows=100 width=97) (actual time=0.052..3.333 rows=100 loops=1)
   ->  Index Scan Backward using acc_abalance_idx on pgbench_accounts  (cost=0.57..13911160.57 rows=400000000 width=97) (actual time=0.050..3.324 rows=100 loops=1)
 Planning Time: 0.388 ms
 Execution Time: 3.347 ms
(4 rows)

Time: 4.220 ms
```

### 分析场景二

这次是，找出3个帐额不为0的储蓄账户。

```sql
select * from pgbench_accounts where abalance != 0 limit 3;
```

在Heap引擎中，explain analyze结果是这样的

```
QUERY PLAN                                                                   
----------
 Limit  (cost=1000.00..7183635.50 rows=1 width=97) (actual time=186459.205..271126.284 rows=3 loops=1)
   ->  Gather  (cost=1000.00..7183635.50 rows=1 width=97) (actual time=186459.203..271126.281 rows=3 loops=1)
         Workers Planned: 8
         Workers Launched: 7
         ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..7182635.40 rows=1 width=97) (actual time=133853.479..254216.098 rows=2 loops=8)
               Filter: (abalance <> 0)
               Rows Removed by Filter: 49971072
 Planning Time: 0.282 ms
 Execution Time: 271126.301 ms
(9 rows)
```

在Hydra引擎中，explain analyze结果是这样的

```
QUERY PLAN                                                                        
----------
 Limit  (cost=0.00..0.30 rows=3 width=97) (actual time=24996.779..24996.782 rows=3 loops=1)
   ->  Custom Scan (ColumnarScan) on pgbench_accounts  (cost=0.00..40151586.02 rows=400001000 width=97) (actual time=24996.777..24996.779 rows=3 loops=1)
         Columnar Projected Columns: aid, bid, abalance, filler
         Columnar Vectorized Filter: (abalance <> 0)
 Planning Time: 7.015 ms
 Execution Time: 25065.955 ms
(6 rows)
```

不等谓词真是万恶之源，PG选择了全表扫描。

让人疑惑的是，Heap引擎中，哪怕有索引也不用。也许是个bug。

而Hydra有自己的定制的扫描和过滤方法。尽管规划的时间较长，但总体执行时间只有Heap引擎的十分之一！

### 分析场景三

这次是一个比较复杂的查询。找出非零帐额账户数量最高的10家银行机构。

```sql
select bid, count(0) as cnt 
from pgbench_accounts
where abalance != 0 
group by bid 
order by cnt desc 
limit 10;
```

在Heap引擎中，explain analyze结果是这样的

```
QUERY PLAN                                                                             
----------
 Limit  (cost=7183635.54..7183635.54 rows=1 width=12) (actual time=256178.105..256178.221 rows=10 loops=1)
   ->  Sort  (cost=7183635.54..7183635.54 rows=1 width=12) (actual time=256178.104..256178.219 rows=10 loops=1)
         Sort Key: (count(0)) DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  GroupAggregate  (cost=7183635.51..7183635.53 rows=1 width=12) (actual time=256175.444..256177.863 rows=3681 loops=1)
               Group Key: bid
               ->  Sort  (cost=7183635.51..7183635.51 rows=1 width=4) (actual time=256175.434..256176.439 rows=9997 loops=1)
                     Sort Key: bid
                     Sort Method: quicksort  Memory: 853kB
                     ->  Gather  (cost=1000.00..7183635.50 rows=1 width=4) (actual time=0.344..256171.733 rows=9997 loops=1)
                           Workers Planned: 8
                           Workers Launched: 7
                           ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..7182635.40 rows=1 width=4) (actual time=132371.762..256165.095 rows=1250 loops=8)
                                 Filter: (abalance <> 0)
                                 Rows Removed by Filter: 49998750
 Planning Time: 0.993 ms
 Execution Time: 256178.292 ms
(17 rows)
```

在Hydra引擎中，explain analyze结果是这样的

```
QUERY PLAN                                                                                      
----------
 Limit  (cost=11820.57..11820.57 rows=1 width=12) (actual time=4849.581..4853.673 rows=10 loops=1)
   ->  Sort  (cost=11820.57..11820.57 rows=1 width=12) (actual time=4849.579..4853.671 rows=10 loops=1)
         Sort Key: (count(0)) DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  GroupAggregate  (cost=11820.54..11820.56 rows=1 width=12) (actual time=4849.074..4853.536 rows=877 loops=1)
               Group Key: bid
               ->  Sort  (cost=11820.54..11820.54 rows=1 width=4) (actual time=4849.062..4853.249 rows=999 loops=1)
                     Sort Key: bid
                     Sort Method: quicksort  Memory: 71kB
                     ->  Gather  (cost=1000.00..11820.53 rows=1 width=4) (actual time=4806.238..4852.964 rows=999 loops=1)
                           Workers Planned: 7
                           Workers Launched: 7
                           ->  Parallel Custom Scan (ColumnarScan) on pgbench_accounts  (cost=0.00..10820.43 rows=57143000 width=4) (actual time=4792.031..4794.666 rows=125 loops=8)
                                 Columnar Projected Columns: bid, abalance
                                 Columnar Vectorized Filter: (abalance <> 0)
 Planning Time: 4.714 ms
 Execution Time: 4862.387 ms
(16 rows) 
```

Hydra只在两个必要的列上使用7个worker执行了并行全列扫描，并使用了向量化过滤器完成过滤，仅仅使用了Heap引擎1.9%的时间，堪称碾压。

## 总结

在OLAP场景中，动辄需要全列分析时：Heap因为行式元组的原因，被迫全表扫描；而Hydra只读取必要的列，而且还是压缩过的，所以在硬盘IO方面有很大优势。

要想在OLAP场景中使用Heap引擎，需要设计良好的索引结构，用空间换时间。

对于存储类似历史记录这样的分析性数据，通常没有OLTP的需求。假如对分析延迟没有强烈的需求，不想增加很多索引，那么可以考虑Hydra引擎，既节省空间，分析性能还还不错。
