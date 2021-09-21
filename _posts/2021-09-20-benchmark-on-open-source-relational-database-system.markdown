---
layout: post
title: "开源关系型数据库的性能对比"
date: 2021-09-20 10:54:03 +0800
categories: 原创
---

围观知乎上PGer和MySQLer互撕了这么多年，其实我一直很想看看这俩面对面较量一下会是什么结果。

最近恰好有个场景，使用关系型数据建模，又要求数据必须强一致，因此非常适合使用关系型数据库来实现。于是，我决定用这个场景来测试一下几个常见数据库的性能表现。

## 数据模型

这个是一个电信领域消费者业务系统的一个场景，主要用于消费者可购买产品的参数管理。消费者端可以看到的各种各样的合约机或者套餐等商品选项（Product Offering）。而这些商品在运营商端则被拆分为多层级的数据结构。不同层级的数据能够相互引用，从而便于运营商管理商品属性。
```
                          ┌──────────────────┐
                          │ product offering │
                          └───┬─────────────┬┘
                              │             │
                 ┌────────────▼───┐   ┌─────▼──────────┐
                 │  product spec  │   │  product spec  ├──────────────┐
                 └┬───────┬───────┘   └───┬───────────┬┘              │
                  │       │               │           │            ┌──▼──────────────┐
            ┌─────▼┐   ┌──▼───┐      ┌────▼─┐        ┌▼─────┐      │  resource spec  │
            │ cfss │   │ cfss │      │ cfss │        │ cfss │      └─────────────────┘
            ├─────┬┘   └┬─────┘      └─────┬┘        └─────┬┘
            │     │     │                  │               │
      ┌─────▼┐    │ ┌───▼──┐              ┌▼─────┐       ┌─▼────┐
      │ rfss │    └─► rfss │              │ rfss │       │ rfss │
      └───┬──┘      └──────┘              └──────┘       └──────┘
          │
          │
┌─────────▼───────┐
│  resource spec  │
└─────────────────┘                                                  
```

如果将这个数据结构看作一棵树，那么可以说：
 - 消费者只能够看到树的根节点（root）
 - 运营商可以修改树的中间节点和叶子节点，包括节点本身的属性以及节点间的指向关系

想要为消费者呈现一个商品（Product Offering）的全部属性，则需要从root节点出发沿着节点的指向关系递归的查询获取所有节点。

## 关系型数据库建模

尽管上面图里列出了5种数据类型，看起来好像可以不同类型数据可以放在不同的表中，但是考虑到：
  - 实际生产环境中数据量并没有那么大（每个运营商有数千数据）
  - 方便业务的变更（新增类型无需再创建数据表）

所以这里把这些数据都存放在一张表里。也就是说每一层级的一个数据节点都在同一张数据表中有对应的一条数据。

这张表的结构如下

```sql
CREATE TABLE specification (
    start_time      timestamp(0) not null,
    end_time        timestamp(0) not null,
    entity_id       binary(16) not null,
    revision_id     binary(16) not null,
    operator_name   varchar(64) not null,
    psr_type        tinyint not null,
    name            varchar(64)  not null,
    description     varchar(2048)  not null,
    properties      json not null,
    PRIMARY KEY(operator_name, entity_id, revision_id)
);
```

请无视各个业务字段。这张表里的核心字段在主键：
  - `operator_name`就是运营商的名字, 
  - `entity_id`是节点数据的id, 
  - `revision_id` revision是entity的历史变更记录，同一时间段内只有一个有效revision

而数据节点的关系是一个N:M的关系，因此使用另一张关系表来表示。

```sql
CREATE TABLE spec_relationship (
    operator_name    varchar(64) not null,
    parent_entity_id binary(16) not null,
    child_entity_id  binary(16) not null,
    PRIMARY KEY(operator_name, parent_entity_id, child_entity_id)
);

CREATE INDEX spec_relationship_p using BTREE ON spec_relationship (operator_name, parent_entity_id);
CREATE INDEX spec_relationship_c using BTREE ON spec_relationship (operator_name, child_entity_id);
```

递归查询的SQL语句则使用`WITH RECURSIVE`来实现。

整体来看，这是一个在小数据集上的递归关系查询。

## 性能测试

软硬件环境 |  |
:--: |  :--:
CPU  |  8C16T 
RAM  |  16GB  
Disk |  NVMe SSD
OS   |  Linux: Debian 5.10.46-4
Docker   | 20.10.5
Postgres | 13.4-buster
MySQL    | 8.0.26
MariaDB  | 10.6.4-focal

这次测试直接使用了Docker Hub上拉取的官方镜像。测试程序是自己用Akka实现的小型性能测试系统。没有用JMeter主要是因为JMeter采用了多线程架构，大并发量测试的时候可能会和数据库内部线程一起争用系统资源。而Akka使用Actor模型，资源消耗更低，吞吐量也很可观。

第一次测试的时候全部使用默认参数，未做调优。不过，MySQL和MariaDB的innodb_buffer_pool_size默认竟然只有128MB，比较影响查询性能。所以第二次将这个值提升到4GB。

测试脚本的行为是，启动时就建立足够多的连接，并准备测试参数集。每一次测试都是全部链接顺序发送同一个查询请求一万次。查询请求的参数会随机变更。每一个场景下会测试三次，并取第三次的数据作为测试结果。

### 测试数据

使用Python脚本向数据库插入模拟数据。预选15个运营商，分别构造4k+数据，整体specification表中的数据有71k+数据。单次查询结果的数据行大致在40～60条的样子。

由于不同运营商的数据不存在交叉引用的可能，所以理论上使用分区表按`operator_name`来分区能够提高查询性能。因为需要检索的数据行和索引都变小了。

而分区或不分区则相当于是两种数据集：一种大一点的数据集71K行，一种小一点的数据集4k行(这个也不完全对，但肯定小一点)。。

### 测试结果


表中数据为：`<p99延迟（ms）> / <p999延迟（ms）> / <max吞吐量（/s）>`


并发连接数        | 10                 | 40                 | 60                  | 80
:--:            |  :--:              | :--:               | :--:                | :--:
Postgres 不分区  | 0.506/1.452/28320  | 0.551/1.793/36907  | 0.550/1.635/36880   | 0.739/3.715/37130
Postgres 分区    | 1.368/2.256/9839   | 1.598/6.806/12672  | 2.980/8.252/11101   | 2.588/7.130/11755
MySQL 不分区     | 1.999/3.815/6808   | 3.381/10.705/7394  | 3.418/7.545/6575    | 3.273/7.313/6537
MySQL 分区       | 1.907/4.467/7660   | 3.110/8.783/8756  |  3.095/6.091/7258    | 3.158/6.954/6957
MariaDB 不分区   | 338.438/348.488/40 | N/A                |  N/A                | N/A
MariaDB 分区     | 0.778/1.157/17040  | 0.871/3.987/22533 |  0.874/3.334/22472   | 1.051/5.798/22410
MariaDB 不分区(调参后) |	0.718/1.103/18368	| 0.764/1.241/24755	| 0.819/1.294/24563	| 1.509/4.903/22461

刚刚看到这个测试结果的时候，可谓是惊掉下巴。

首先Postgres在分区后反而性能大幅下降。估计PG的分区表是针对大数据集设计的，71k+这种其实也不大，没必要搞分区表。在不分区的情况下，PG的吞吐量足足超越了MariaDB最好状态的50%。这个简直了！！！

然后是MariaDB在不分区的情况下性能已经失去水准，所以后面在增加并发连接数的时候没有去测。在小数据集查询场景下，随着并发连接数的持续增加，吞吐量能够一直保持稳定。这点也值得称赞。

MariaDB默认innodb_buffer_pool_size只有128MB，这个对大数据集查询场景影响蛮大的。当参数调整为4GB之后，延迟吞吐均达到合理水平。可以看出在并发连接为80的时候吞吐出现小幅度下降。

而对于MySQL，从延迟数据来看数据集的大小并没有影响到MySQL的表现。但是这吞吐量不上1w是什么鬼，和前面二位在两个次元吗？MySQL的默认innodb_buffer_pool_size也是128MB。但提升这个数值并没有带来性能上的改变。观察测试期间的系统资源占用情况，PG和MariaDB在大负载的情况下16颗逻辑线程占用率均能达到95+%，而MySQL则始终保持在70%多，不到80%的水准。显然MySQL的community版本里在刻意压制其性能表现。呵呵，看来Oracle眼里当真全都是生意。不过MySQL的能力还是在的，使用默认参数在大数据集的查询中没有像MariaDB那样完全失去水准。此外，MySQL在60并发连接的时候就开始出现吞吐下降，这拐点来的也太早了！Postgres在80并发连接的时候吞吐量甚至还有小幅上涨。

再对比一下，数据集的磁盘占用情况，Postgres是430M，MariaDB是673M，MySQL是928M。

总体来看，Postgres完胜！“The World's Most Advanced Open Source Relational Database”完全是实至名归！

MariaDB提高innodb_buffer_pool_size之后，所有场景表现均合理。但是吞吐量依然落后于Postgres。不过至少比MySQL社区版强多了。

MySQL是个玩具。会玩的人打开性能封印可以愉快的玩耍，不会玩的只能被Oracle羞辱了。
