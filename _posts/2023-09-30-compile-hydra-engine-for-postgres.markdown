---
layout: post
title: "Postgres列式引擎hydra的编译安装"
date: 2023-09-30 11:16:39 +0800
categories: 原创
---

[Hydra](https://www.hydra.so/)是一款为Postgres设计开发的列式存储引擎，以AGPL协议开源，主要针对的是OLAP分析性场景。

虽然Hydra源代码开源，但是官方只提供了两种使用方式：
- 公有云平台的收费Image，主要用来做商业服务
- 容器镜像，主要做开发测试性质的任务

要想安装在自己服务器运行的PG里，还是需要手动编译的。这里我的PG是自己编译安装在本地的14.9版本。

首先拉一下hydra的源码仓

```
git clone https://github.com/hydradatabase/hydra.git
cd hydra
```

然后在hrdya的repo根目录下执行

```
cd /columnar 
./configure
```

如果有一些依赖没有安装，configure会报错。那就安装一下

```
# 我用的是openSUSE，使用Debian系或者RHEL系的自行参考apt/yum包管理系统的使用方法
sudo zypper in libcurl-devel liblz4-devel libzstd-devel 
```

然后正式编译安装

```
cd src/backend/columnar 
sudo make install
```

注意在最后一步，Makefile依赖于pg_config读取extension的安装目录，但是pg_config的路径被固定为`/usr/bin/pg_config`。如果这个文件不存在，可以创建一个软链接。

```
sudo ln -s `which pg_config` /usr/bin/pg_config
```

安装完成后，需要修改PG的配置文件`postgresql.conf`, 在shared_preload_libraries里增加`columnar.so`。

接着，重启PG服务。如果是用systemd管理的服务，可以执行

```
sudo systemctl restart postgresql
```

重启完成后，进入控制台

```
psql -U postgres
```

在控制台里重建扩展

```
postgres=# create extension columnar;
CREATE EXTENSION
```

如果这一步没有任何报错，那就可以开始创建表了。例如

```
postgres=# create table test(id bigint, info text) using columnar;
CREATE TABLE
```

然后就可以愉快的玩耍了！
