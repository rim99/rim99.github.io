---
layout: post
title: 两个PostgreSQL和Spring Data JPA之间的坑
date: 2018-02-18 19:53:02
categories: 原创
---

今天在家调试Spring Data JPA，打算研究一下JPA的黑技巧，结果出师不捷。因为家里用的数据库是PostgreSQL，和公司的MySQL有点区别。

这里把今天遇到的问题记录一下。

首先，配置好POM，使用默认配置启动的时候会报一个错误。

```
java.sql.SQLFeatureNotSupportedException: Method org.postgresql.jdbc.PgConnection.createClob() is not yet implemented.
```

根据[网上查到的资料](http://vkuzel.blogspot.com/2016/03/spring-boot-jpa-hibernate-atomikos.html)，这个是由于：Hibernate尝试验证PostgreSQL的CLOB特性，但是PostgreSQL的JDBC驱动并没有实现这个特性，所以抛出了异常。

解决方法是：关闭这个特性的检测。在配置文件里添加如下内容：

```yaml
spring:
  jpa:
    # Disable feature detection by this undocumented parameter. Check the org.hibernate.engine.jdbc.internal.JdbcServiceImpl.configure method for more details.
    properties.hibernate.temp.use_jdbc_metadata_defaults: false
    # Because detection is disabled you have to set correct dialect by hand.
    database-platform: org.hibernate.dialect.PostgreSQL9Dialect
```

接下来，服务可以正常启动了。但是在存表的时候却有一个报错:

```
org.postgresql.util.PSQLException: ERROR: relation "hibernate_sequence" does not exist
```

这个报错的原因是领域类里面设定了主键为自增类型，标记了注解`@GeneratedValue`。这个注解的默认自增类型是`GenerationType.AUTO`。如果注解的自增类型是`AUTO`，对于PostgreSQL，Hibernate会默认识别为`GenerationType.SEQUENCE`。这时就需要一张序列表。如果Hibernate找不到指定的序列表或者没有指定序列表，就会查找名为`hibernate_sequence`的序列表。如果数据库里没有这张序列表，就会报上面这个错。

不像MySQL的列可以有`AUTO_INCREMENT`属性表示自增，PostgreSQL默认使用的是序列`sequence`来实现类似功能。`sequence`是一种特殊的单行表，用于生成序列数字。

看这里：["hibernate_sequence" does not exist](https://coderanch.com/t/487173/databases/hibernate-sequence-exist)。

哎，本来一张表的事情，要用两张表来完成，好麻烦～～～

不过，PostgreSQL已经完善了这个问题，提供一种“伪“类型——`serial`。在建表的时候将主键声明为此类型，然后将注解的自增类型修改为`GenerationType.IDENTITY`，问题就解决了。

为什么说是“伪“类型呢？[PostgreSQL的文档](https://www.postgresql.org/docs/9.6/static/datatype-numeric.html#DATATYPE-SERIAL)说的很清楚。

建表时候，使用语句

```sql
CREATE TABLE tablename (
    colname SERIAL
);
```

其实就相当于

```sql
CREATE SEQUENCE tablename_colname_seq;
CREATE TABLE tablename (
    colname integer NOT NULL DEFAULT nextval('tablename_colname_seq')
);
ALTER SEQUENCE tablename_colname_seq OWNED BY tablename.colname;
```

完全可以把`serial`看作是PostgreSQL的语法糖。













