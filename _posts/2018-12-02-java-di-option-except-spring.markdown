---
layout: post
title: 除了Spring，还有什么DI选择
date: 2018-12-02 15:13:53
categories: 原创
---

在Java Web世界里，Spring如同基石一般的存在。

不过，Spring是一个重量级框架。假如项目仅仅需要用到的是Spring提供的DI依赖注入和AOP面向切面编程两个功能。那么，其实还有一些轻量级的框架可以选择。

# HK2

HK2是JavaEE规范 `JSR-330`的官方实现，是一个轻量级的动态依赖注入框架。

不过虽然背景看起来很不错，但是目前看到的HK2的应用也就是Jersey——另一个JavaEE规范的官方实现，是一个实现了JAS-RS 2.0规范的轻量级Web路由框架，作用类似于Spring MVC，但是比后者性能要好一些。

HK2有一个比较奇特的地方是，用户允许不向`ServiceLocator`——一个类似于`ApplicationContext`的角色——预先注册对象。`ServiceLocator`的`create(Class<T> clazz)`对象方法可以动态的分析`clazz`类的依赖关系，然后调用合适的构造器生成`clazz`类对象。当然也可以预先注册关系。

总归是小众框架，HK2同时维护了与Spring、Guice两个DI框架的桥接器，以允许不同的框架之间共享对象。

# Guice

Guice是一个轻量级的动态依赖注入框架，目前由Google在维护。

与Spring使用XML或者注解完成自动注入不同，Guice需要将interface和concrete class的对应关系写在Module中，也就是使用代码作为配置文件。配置文件类Module可以用来生成统一的对象工厂Injector，类似于Spring的ApplicationContext。

简单的看看Injector的源码可以发现，Guice将不同的对象成员的依赖注入使用不同的MemberInjector来实现。这些细粒度的MemberInjector通过延迟加载的方式来生成，缓存在一个HashMap中。

由于延迟加载的原因，使用Guice的项目的启动速度一般会更快。对比一下，一个没有任何业务代码的Spring Boot项目启动时间要花7到10秒。因为Spring Boot预先生成所有的Bean单例，然后按需供应。如果遇到需要按需生成新对象的场景，Spring会成为性能的拖累。一份[比较久远的数据](https://www.javalobby.org//articles/guice-vs-spring/)是**Spring 2.5构造对象的耗时是Guice 1.0的10倍之多**。不知道现在怎么样了，将来有空测一测。

而Guice则是默认每次生成一个新对象。一个由此而来的另一个好处是，不需要过多的关心线程安全的问题，因为每次都会生成新的对象。

HK2、Guice都有一个特点，如果一个依赖无法注入，会直接抛异常；而Spring会传递空指针，最终在代码的其他位置报错，不利于debug。

此外，Guice的生态也还不错。大致搜了一下，MyBatis、Shiro都有官方支持。而且，Spring也还维护了一个与Guice的桥接器，方便接入Spring的生态圈。

# Dagger2

Dagger2也是Google在维护。

与前两者不同，Dagger2是一个静态的依赖注入框架。所有依赖都在编译期完成注入，因此性能很好。Google维护Dagger2主要想为Android生态圈提供一个高性能的DI框架，有助于降低功耗、降低app的响应时间，以提高用户体验。

如果需要，Dagger2也可以使用Lazy范型类包装一些需要延迟加载的对象。这些对象可以在第一次被调用的时候，再完成初始化。所以，Dagger2也可以和Guice一类的动态依赖注入框架配合使用。

# 简单总结

总的来讲，除了Spring之外，也就Guice的替代性更高一些。

Guice框架使用代码作为配置，使得依赖的来源很直观。而Spring框架在这一点，做得比较隐晦。

同时包括Spring在内，上述所有框架都支持JSR-330规范。因此，为了将来可能的更换DI框架考虑，实践中应该尽量多的使用JSR-330规范提出的注解。

目前，自己在试着使用Jersey、Jetty和Guice实现一个[没有Spring的微服务系统](https://github.com/ZhangXinJason/account-demo)。将来如果有机会，会试一试Dagger2框架。

# 参考资料

* [Introduction to HK2](https://javaee.github.io/hk2/introduction.html)
* [google/guice](https://github.com/google/guice)
* [Dagger2](https://google.github.io/dagger/)
