---
layout: post
title: "块存储与对象存储"
date: 2025-07-19 20:37:17 +0800
categories: 原创
---

云服务商在存储方面大致都会提供这两种方案满足存储需求。

块存储，顾名思义，能够提供文件系统块级别的IO能力，用起来很像硬盘设备。之所以说“像”，是因为块存储本质上是一种服务。其背后是庞大的服务器集群，依托海量硬盘阵列，面向用户使用的虚拟机设施，提供网络服务，其网络协议类似于iSCSI。虚拟机内部有相应的驱动文件，把这种网络服务能力映射成本地块设备，供用户使用。而在块存储服务器集群内，用户数据会被存储多份，满足高可用。通常，各家云服务商都有SLA协议确保其服务能力。

对象存储，只存储文件级别的内容。甚至不能建立文件夹体系，只能通过文件名前缀模拟文件夹结构。所以，像MySQL数据库这样通常需要直接操纵文件块的软件，不能直接使用对象存储。

|               | 块存储                                       | 对象存储                                                  |
|---------------|---------------------------------------------|----------------------------------------------------------|
| **存储粒度**   | 文件系统块级别                                 | 文件级别                                                  |
| **访问方式**   | 类似硬盘设备，支持随机读写                       | 通过HTTP API访问，适合顺序读写                              |
| **高可用性**   | 数据多副本存储，高可用，数据可用性多为4个9或5个9    | 数据多副本或纠删码保护，高可用，数据可用性多为11个9或12个9       |
| **典型服务**   | AWS EBS, Azure Disk Storage                  | AWS S3, Azure Blob Storage                              |

对象存储有两个很明显的优势，那就是成本和吞吐能力。在同等使用空间下，对象存储成本更低。块存储每GB价格更高，而且需要预置更大空间，所以整体价格很贵。

以宁夏区AWS EBS和S3为例，假设需要存储800GB内容，块设备预置1TB空间：

| 服务              | 容量         | IOPS              | 吞吐量                                  | 定价示例      |
|-------------------|-------------|-------------------|----------------------------------------|-------------|
| AWS EBS / GP3     | 预置1TB      | 3000              | 125MBps                                | 每月￥543.95 |
| AWS EBS / SC1     | 预置1TB      | 12（基本）/80（突发）| 12MBps（基本）/80MBps（突发）            | 每月￥101.99  |
| AWS S3 / 标准类型  | 实际使用800GB | 不适用（对象存储）   | 支持多连接并行读取文件对象，每连接85-90MBps | 每月￥140.40  |

AWS S3对吞吐能力没有设置直接的上限。客户端可以一直增加并发连接，直到吞吐能力受到EC2的网卡参数限制。

此外，EBS还有一个隐藏的问题。用户使用EBS作为存储设施，假如需要多AZ副本，只能先将其挂载在EC2虚拟机上，通过虚拟机进程实现数据传递。这一环节实际上存在跨AZ网络流量，是需要另外付费的。而S3上的文件对象，内部实现多AZ副本，不需要另外付费。

云服务商对块存储与对象存储各自设定了不同的性能优势，和特定的成本差异。这实际上会显著地改变云上存储系统的形态。

过去分布式数据库系统都会使用典型的多数据副本架构，例如ElasticSearch， Cassandra。而RDBMS数据库系统，例如PostgreSQL使用物理复制能力，建立一写多读的数据库集群。

这种架构很适合传统机房服务器。传统物理服务器的硬盘系统，相比云服务商块存储更容易进入维护期。一旦硬盘发生损坏，更换硬盘和重建RAID阵列等需要花费比较长的时间。在这段时间内，数据库系统整体的服务能力，因为数据多副本，依然得到了保证。而且，传统物理服务器的硬盘价格往往比云服务商的块存储要便宜很多。多副本的成本开销不明显。

如果考虑云服务块存储，每一个块存储资源本身为了可用性保证已经实现了多副本的。在此基础上，继续多副本，冗余很明显。成本也会根据复制因数成倍上升的。而块存储层面多副本的收益，却不明显。因为云服务商的块存储不怎么出错。此外，数据库中有很多归档性质的数据并不需要实时的读写。这部分数据放在块存储资源上多少有些浪费。

在未来云计算生态里，块存储应该成为整个数据库系统的写入缓冲或读取缓存，维持小规模存在。大部分数据应该保留在对象存储中，充分利用其跨AZ免费复制的能力。数据写入与读取都是近似Serverless那样按需运行的方式。读写请求的处理能够并行计算，充分利用对象存储并行连接带来的高吞吐优势。如此，既降低成本，还能保持足够高的性能。

顺着这个趋势发展的产品有（不完全列举）：
- [AutoMQ](https://github.com/AutoMQ/automq)
- Grafana的LGTM栈，以[Loki](https://grafana.com/docs/loki/latest/get-started/architecture/)为例
- [CortexMetrics](https://cortexmetrics.io/docs/architecture/)
- [Thanos](https://thanos.io/tip/thanos/design.md/#architecture)
- [InfluxDB](https://docs.influxdata.com/influxdb3/clustered/reference/internals/storage-engine/)
- [Postgres Neon](https://neon.com/docs/introduction/architecture-overview)
- [Snowflake](https://docs.snowflake.com/en/user-guide/intro-key-concepts)
- [Databend](https://docs.databend.com/guides/products/dc/architecture)

还有一个特别的数据库引擎，Rockset和OpenAI维护的RocksDB分支：[rocksdb-cloud](https://github.com/rockset/rocksdb-cloud)不但可以利用本地文件系统读写WAL，还能利用Apache Kafka和AWS Kinesis读写WAL，而刷盘的SSTable文件则写入S3。这个思路完美地实现了存算分离。假如引擎进程崩溃，我们可以在不同的位置另起进程，通过读取Kafka/Kinesis里的WAL数据恢复Memtable，而SSTable文件则在S3 bucket里没有变化。从而，引擎进程可以不依赖原来的环境继续运行。

## 参考资料

- [Features of Amazon EBS](https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html#ebs-overview)
- [Amazon EBS General Purpose SSD volumes](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html)
- [Amazon EBS Throughput Optimized HDD and Cold HDD volumes](https://docs.aws.amazon.com/ebs/latest/userguide/hdd-vols.html)
- [Performance design patterns for Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-design-patterns.html)

