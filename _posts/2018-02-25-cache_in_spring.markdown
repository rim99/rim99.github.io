---
layout: post
title: Spring的缓存
date: 2018-02-25 19:53:02
categories: 翻译
---


> 本文摘译自[Spring Framework Document](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache)

在Spring框架中使用缓存，需要实现`org.springframework.cache.Cache`和`org.springframework.cache.CacheManager`两个接口。前者定义了缓存操作的基本方法，例如获取，更新，剔除等操作；后者负责管理Cache对象。

Spring框架提供了几个注解方便缓存的配置：

* `@Cacheable` - 开启缓存
* `@CachePut` - 强制更新缓存
* `@CacheEvict` - 清理缓存
* `@Caching` - 将上述多条注解合并在一起
* `@CacheConfig` - 类级别缓存注解

# `@Cacheable` - 开启缓存

这是一个方法注解，表示方法的结果会被缓存。

```Java
@Cacheable({"books", "isbns"})
public Book findBook(ISBN isbn) {...}
```

`@Cacheable`默认参数为`cacheNames`，表示缓存对象的id，可以是1个id，也可以是1组id数组。

## 缓存键值

默认的缓存键值为`org.springframework.cache.interceptor.SimpleKey`对象。

* 如果方法没有参数，那么键值就是`SimpleKey.EMPTY`；
* 如果只有一个参数，那么键值就是这个参数；
* 如果有多个参数，那么键值就是以方法的全部参数为构造参数的`SimpleKey`对象。

缓存键值由`org.springframework.cache.interceptor.KeyGenerator`接口实现，默认实现方法是`org.springframework.cache.interceptor.SimpleKeyGenerator`。

`@Cacheable`注解的`key`属性的值为SpEL表达式，为缓存键值的生成方法；也可以使用`keyGenerator`属性，使用自定义的键值生成器。

## 线程同步

如果有多条线程发起同样的参数调用，Spring可以保证只有一个调用被实际执行，其余调用则直接等待结果。

开启方法是`sync`属性设置为`true`。

```Java
@Cacheable(cacheNames="foos", sync=true)
public Foo executeExpensiveOperation(String id) {...}
```

## 设置缓存条件

可以为`condition`和`unless`两个属性赋值，值类型为SpEL表达式，来实现部分条件下缓存方法结果。

```Java
@Cacheable(cacheNames="book", condition="#name.length() < 32")
public Optional<Book> findBook(String name)
```

这个例子表示`name`参数长度大于32的时候，调用结果不会缓存。

需要注意的是，`condition`是先决条件，在方法调用前判断，而`unless`则在方法调用后才会判断。

# `@CachePut` - 强制更新缓存

@CachePut`的用法和`@Cacheable`类似。

# `@CacheEvict` - 清理缓存

`@CacheEvict`可以清除Cache对象的部分内容，也可以设置`allEntries`属性为`true`从而清除Cache对象的全部内容。

通过设置`beforeInvocation`属性是否为`true`可以决定是否在方法执行前清理缓存。

# `@Caching` - 合并注解


```Java
@Caching(evict = { @CacheEvict("primary"), @CacheEvict(cacheNames="secondary", key="#p0") })
public Book importBooks(String deposit, Date date)
```

# `@CacheConfig` - 类级别缓存注解

`@CacheConfig`作用在类定义上，方便整合类中各方法的重复配置。

```Java
@CacheConfig("books")
public class BookRepositoryImpl implements BookRepository {

    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```

总体来讲，一个方法上的缓存配置，自上而下，有三个定义层面：

1. `CacheManager`, `KeyGenerator`的全局配置
2. `@CacheConfig`的类定义层级注解
3. 方法层级注解

# 缓存开启

在某一个`@Configuration`类中，加上`@EnableCaching`注解。
















