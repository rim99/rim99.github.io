---
layout: post
title: "让Postgres集群再次Q弹"
date: 2022-07-24 10:57:34 +0800
categories: 原创
---

建设可水平扩展的数据库系统一直是业界发力的主要方向之一。近年来各种新生NewSQL数据库层出不穷，解决的痛点之一就是水平扩展。与此同时，老牌的Postgres也没有闲着。近期即将GA的Postgres 15就增加了一项新功能，可以很好地改善水平扩展的问题。

## 过去的方式

分布式系统离不开数据复制功能，数据库系统也不例外。Postgres支持两种复制方式：
* 流式复制（Stream Replication）:基于WAL日志的复制方式。相比更早以前的WAL日志文件复制，复制延迟更低。
* 逻辑复制（Logical Replication）：相比流式复制，可以做到更小粒度的数据复制。截止14大版本，已经可以做到表级复制。

表级逻辑复制是有一些不足的。

Postgres里每张物理数据表、索引在大小不超过1GB的时候，会放置在单独的文件中。因此，在读取逻辑全表记录的时候，如果没有很好的条件来限制逻辑表范围，那么一次查询会涉及多个物理表，从而读取多个文件，增加了时间开销。

此外还有一个数据再分配的问题：当一个单个物理表过大，需要数据拆分的时候，系统得再创建新的物理表，再做数据复制。比较麻烦。

## Postgres 15逻辑复制的行级过滤器

在Postgres 15中，逻辑复制增加了一个新功能-行级过滤器（Row Filter），可以实现亚表级复制，可以很好地解决上述问题。

Postgres的逻辑使用的是订阅发布模型。这个行级过滤器就是在发布端设置谓词，在对应的slot里面是写入符合条件的数据行的修改记录。

其优势就是：

首先，数据不必分表记录。只要在同一张表上，设置不同的发布端，就实现数据不同方向的复制了。

其次，数据拆分的时候，不再需要复制表数据。只要新增发布端就可以，就可以增加复制体系。

更方便了。

## 举个例子

### 准备一个小集群

首先用Docker-Compose启动两个Postgres 15进程起来：

```
version: "3.3"

services:
  pg1:
    image: postgres:15beta2-alpine3.16
    volumes:
      - ./_data/pg1:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
      - ./postgresql.conf:/opt/postgresql.conf
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    command:
      - "postgres"
      - "-c"
      - "config_file=/opt/postgresql.conf"
  pg2:
    image: postgres:15beta2-alpine3.16
    volumes:
      - ./_data/pg2:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
      - ./postgresql.conf:/opt/postgresql.conf
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    command:
      - "postgres"
      - "-c"
      - "config_file=/opt/postgresql.conf"
```

其中，init文件中包含一个建表语句，就是此次测试用表

```
CREATE TABLE employee (
    id          uuid not null,
    company     text not null,
    first_name  text not null,
    last_name   text not null,
    mobile      text not null,
    PRIMARY KEY(id)
);
```

而postgresql.conf包含以下设置

```
listen_addresses = '*'
wal_level = logical
```

### 启动

```
docker-compose up -d
```

启动完成后，打开各自的psql进程。

```
docker exec -it elastic-postgres-prototype_pg1_1 /bin/bash -c "psql postgres postgres"
docker exec -it elastic-postgres-prototype_pg2_1 /bin/bash -c "psql postgres postgres"
```

### 创建发布端

在pg1中执行命令，创建复制的发布端

```
CREATE PUBLICATION employee_company_a_c FOR TABLE employee WHERE (company = 'Company A' or company = 'Company C');
```

同样地，在pg2中执行

```
CREATE PUBLICATION employee_company_b_d FOR TABLE employee WHERE (company = 'Company B' or company = 'Company D');
```

### 创建订阅端

在pg1中执行命令，创建复制的订阅端

```
CREATE SUBSCRIPTION employee_company_b_d
    CONNECTION 'host=pg2 port=5432 user=postgres dbname=postgres password=postgres'
    PUBLICATION employee_company_b_d;
```

同样地，在pg2中执行

```
CREATE SUBSCRIPTION employee_company_a_c
    CONNECTION 'host=pg1 port=5432 user=postgres dbname=postgres password=postgres'
    PUBLICATION employee_company_a_c;
```

### 在pg1中插入数据

向pg1中插入数据：

```
INSERT INTO employee (
    id,
    company,
    first_name,
    last_name,
    mobile
) VALUES (
    'a5b48d37-80b8-4b1a-887c-7a98796e0c53',
    'Company A',
    'Marley',
    'Tanner',
    '56608087122'
);

INSERT INTO employee (
    id,
    company,
    first_name,
    last_name,
    mobile
) VALUES (
    '1b95511b-1003-40eb-a6ff-0b70085f2d6d',
    'Company C',
    'Ruth',
    'Coleman',
    '462189036862'
);
```

这时，我们在pg1和pg2中查看employee表，发现两个数据都出现了。

```
postgres=# select * from employee;
                  id                  |  company  | first_name | last_name |    mobile
--------------------------------------+-----------+------------+-----------+--------------
 a5b48d37-80b8-4b1a-887c-7a98796e0c53 | Company A | Marley     | Tanner    | 56608087122
 1b95511b-1003-40eb-a6ff-0b70085f2d6d | Company C | Ruth       | Coleman   | 462189036862
(2 rows)
```

### 在pg2中插入数据

向pg2中插入数据：

```
INSERT INTO employee (
    id,
    company,
    first_name,
    last_name,
    mobile
) VALUES (
    '4b7181ef-8253-45b3-bbb8-5a1c13c9b565',
    'Company B',
    'Jordan',
    'Samson',
    '33263758190'
);

INSERT INTO employee (
    id,
    company,
    first_name,
    last_name,
    mobile
) VALUES (
    '59168cc9-1db5-43c2-93d8-b45a7f6365c0',
    'Company D',
    'Elizabeth',
    'Nash',
    '93910372855'
);
```

这时，我们在pg1和pg2中查看employee表，发现所有数据在各自Postgres实例中都可以看得到。

```
postgres=# select * from employee;
                  id                  |  company  | first_name | last_name |    mobile
--------------------------------------+-----------+------------+-----------+--------------
 a5b48d37-80b8-4b1a-887c-7a98796e0c53 | Company A | Marley     | Tanner    | 56608087122
 1b95511b-1003-40eb-a6ff-0b70085f2d6d | Company C | Ruth       | Coleman   | 462189036862
 4b7181ef-8253-45b3-bbb8-5a1c13c9b565 | Company B | Jordan     | Samson    | 33263758190
 59168cc9-1db5-43c2-93d8-b45a7f6365c0 | Company D | Elizabeth  | Nash      | 93910372855
(4 rows)

```

### 数据拆分

举个例子，假如数据表中`company`为`Company A`的数据行特别多，需要拆分。那么可以pg1中新增两个发布端，其谓词分别设置为`company = 'Company A'`和`company = 'Company C'`。在新的pg3中订阅``company = 'Company A'`.

当数据复制完成后，在pg3中创建对应的发布端。从机pg2，可以转而订阅pg3的发布端。而pg1上的相应数据和发布端就可以删除了。

### 总结

在上面这个例子中，我们实现了一个简单的双主Postgres集群。数据表employee按照`company`字段分为两个shard。各自有两个replica，为一主一从模式。

而数据拆分后，实现了一个简单的三主集群。employee数据表按照`company`字段分为三个shard。各自有两个replica，为一主一从模式。

按照需求，也可以实现每shard有3个replica，保证每一个Postgres实例都可以读取全表数据，而分担写入压力。

## 参考资料

* [Chapter 31. Logical Replication - 31.3. Row Filters](https://www.postgresql.org/docs/15/logical-replication-row-filter.html#LOGICAL-REPLICATION-ROW-FILTER-RULES)
* [PostgreSQL指南：内幕探索](https://book.douban.com/subject/33477094/)
