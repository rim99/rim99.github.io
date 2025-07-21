---
layout: post
title: "块存储与对象存储"
date: 2025-07-19 20:37:17 +0800
categories: 原创
---

云服务商在存储方面大致都会提供(但不限于)这两种方案满足用户需求。

块存储，顾名思义，能够提供原始块设备级别的IO能力，用起来很像硬盘设备。之所以说“像”，是因为块存储本质上是一种服务。其背后是庞大的服务器集群，依托海量硬盘阵列，面向用户使用的虚拟机设施，提供网络服务，其网络协议类似于iSCSI。虚拟机内部有相应的驱动文件，把这种网络服务能力映射成本地块设备，供用户使用。而在块存储服务器集群内，用户数据会被存储多份（大多在同一AZ内），满足高可用。通常，各家云服务商都有SLA协议确保其服务能力。

对象存储，以HTTP RESTful API提供服务，只能存储对象，属于文件级别的内容，甚至不能建立文件夹体系，而是通过文件名前缀来模拟文件夹层级结构。所以，像MySQL数据库这样依赖POSIX API，需要直接操纵文件系统块的软件，不能直接使用对象存储。

|               | 块存储                                       | 对象存储                                                  |
|---------------|---------------------------------------------|----------------------------------------------------------|
| **存储粒度**   | 文件系统块级别                                 | 文件级别                                                  |
| **访问方式**   | 类似硬盘设备，支持随机读写                       | 通过HTTP API访问，适合顺序读写                              |
| **高可用性**   | 数据多副本存储，服务可用性多为4个9，数据耐久性多为4个9或5个9    | 数据多副本或纠删码保护，服务可用性多为4个9，数据耐久性多为11个9或12个9       |
| **典型服务**   | AWS EBS, Azure Disk Storage                  | AWS S3, Azure Blob Storage                              |

对象存储有两个很明显的优势，那就是成本和吞吐能力。在同等使用空间下，对象存储成本更低。对于愿意为低频读写付费的数据，如果使用S3 Glacier，长期存储成本可以更低。而块存储每GB价格更高，而且需要预置更大空间，所以整体价格通常较高。

以宁夏区AWS EBS和S3为例，假设需要存储800GB内容，块设备预置1TB空间：

| 服务              | 容量         | IOPS              | 吞吐量                                  | 定价参考（具体查询官方现价）|
|-------------------|-------------|-------------------|----------------------------------------|-------------|
| AWS EBS / GP3     | 预置1TB      | 3000              | 125MBps                                | 每月￥543.95 |
| AWS EBS / SC1     | 预置1TB      | 12（基本）/80（突发）| 12MBps（基本）/80MBps（突发）            | 每月￥101.99  |
| AWS S3 / 标准类型  | 实际使用800GB | 不适用（对象存储）   | 支持多连接并行读取文件对象，每连接85-90MBps（理论值） | 每月￥140.40  |

AWS S3对吞吐能力没有设置直接的上限。客户端可以一直增加并发连接，直到吞吐能力受到EC2的网卡参数限制。

当然，S3在性能方面也有一些不足。一个明显的缺点就是，延迟偏高。例如，对于S3标准服务，读取请求的响应首字节到达客户端的延迟在百毫秒级别。这使得直接基于S3实现OLTP会非常困难。相比之下，EBS块存储能够实现亚毫秒级延迟，对于实时读写的需求来说很合适。

EBS有一个隐藏的问题。用户使用EBS作为存储设施时，假如需要实现跨AZ多副本，只能先将其挂载在EC2上，通过虚拟机进程实现数据复制。这一环节实际上存在跨AZ网络流量，是需要另外付费的。而主流的对象存储服务能够实现跨AZ多副本，不需要额外付费。

云服务商对块存储与对象存储各自设定了不同的性能优势，和特定的成本差异。这实际上会显著地改变云上存储系统的形态。

过去分布式数据库/存储系统都会使用典型的多数据副本架构，例如ElasticSearch， Cassandra，HDFS等。而RDBMS数据库系统，例如PostgreSQL使用物理复制能力，可以建立一写多读的数据库集群。

这种架构很适合传统机房服务器。传统物理服务器的硬盘系统，相比云服务商块存储更容易进入维护期。一旦硬盘发生损坏，更换硬盘和重建RAID阵列等需要花费比较长的时间。在这段时间内，数据库系统整体的服务能力，因为数据多副本，依然得到了保证。而且，传统物理服务器的硬盘价格往往比云服务商的块存储要便宜很多。多副本的成本开销不明显。

如果考虑云服务块存储，每一个块存储资源本身为了可用性保证已经实现了多副本的。在此基础上，继续多副本，冗余很明显。成本也会根据复制因数成倍上升的。而块存储层面多副本的收益，相比较而言，不够明显。因为云服务商的块存储不怎么出错。_当然了，块存储很少出错，不代表应用进程不会出错。保持数据跨AZ多副本在另一个层面仍然有积极意义。_

此外，数据库中有一些归档性质的数据，其实并不需要实时的读写。这部分数据放在块存储资源上多少有些浪费。

总的来说，在云计算生态里，对于实时读写的场景，例如OLTP，块存储依然非常合适。而对于分析性需求，利用好对象存储则很有意义。例如，让块存储成为整个数据库系统的写入缓冲或读取缓存，维持小规模存在。大部分数据应该保留在对象存储中，充分利用其跨AZ多副本无需额外收费的特点。数据写入与读取都是近似Serverless那样按需运行的方式。读写请求的处理能够并行计算，充分利用对象存储并行连接带来的高吞吐优势。如此，既降低成本，还能保持足够高的性能。

顺着这个趋势发展的产品有（不完全列举）：
- [AutoMQ](https://github.com/AutoMQ/automq)（消息队列）
- Grafana的LGTM栈，以[Loki](https://grafana.com/docs/loki/latest/get-started/architecture/)为例 （时序性数据）
- [CortexMetrics](https://cortexmetrics.io/docs/architecture/)（时序性数据）
- [Thanos](https://thanos.io/tip/thanos/design.md/#architecture)（时序性数据）
- [InfluxDB](https://docs.influxdata.com/influxdb3/clustered/reference/internals/storage-engine/)（时序性数据）
- [Postgres Neon](https://neon.com/docs/introduction/architecture-overview)（**OLTP**）
- [Snowflake](https://docs.snowflake.com/en/user-guide/intro-key-concepts)（OLAP）
- [Databend](https://docs.databend.com/guides/products/dc/architecture)（OLAP）

还有一个特别的数据库引擎，Rockset和OpenAI维护的RocksDB分支：[rocksdb-cloud](https://github.com/rockset/rocksdb-cloud)可以利用本地文件系统读写WAL，而刷盘的SSTable文件则写入S3。

最后，对象存储这一方面，各家厂商也在不断的推出新的产品。例如AWS S3 Express One Zone，Google Cloud Rapid Storage，这些只在一个AZ中保留数据，但能够提供更低延迟，更高并发吞吐的服务（S3 EOZ甚至比EBS价格更高），对数据库生态将会带来什么影响，值得期待。

## 参考资料

- [Features of Amazon EBS](https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html#ebs-overview)
- [Amazon EBS General Purpose SSD volumes](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html)
- [Amazon EBS Throughput Optimized HDD and Cold HDD volumes](https://docs.aws.amazon.com/ebs/latest/userguide/hdd-vols.html)
- [Performance design patterns for Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-design-patterns.html)
- [Optimizing S3 Express One Zone performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-express-performance.html)

