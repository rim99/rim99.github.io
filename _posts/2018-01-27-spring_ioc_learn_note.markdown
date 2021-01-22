---
layout: post
title: Spring IOC容器学习笔记
date: 2018-01-27 19:53:02
categories: 笔记
---

>Spring框架的基石是core模块。core模块主要实现了IOC和AOP两大功能。
>
>其中IOC（iversion-of-control），也就是控制反转，有时也称作依赖注入（DI， dependengcy-injection），指的是，将对象的生命周期委托给第三方容器来管理，降低代码耦合度。
>
>AOP（aspect-oriented-programming），面向切面编程，是在动态代理的基础上实现更高层级的抽象封装。
>
>最近在看IOC的文档，稍作总结，方便以后查看。


`org.springframework.beans.factory.BeanFactory`就是IOC容器的接口定义。一个BeanFactory对象就是一个容器。而`org.springframework.context.ApplicationContext`在BeanFactory的基础上进行了封装，可以实现Spring Bean对象的生命周期管理、监听事件等功能。

Spring主要支持基于XML、注解、Java等的三种配置方式，来管理对象。容器会将对象加载为Bean对象。也支持Groovy脚本来加载Bean对象。

按默认方式，一个类在同一个IOC容器中只有一个实例，因此在一个IOC容器中，可以视作单例模式。当然了，不同的容器中，相同类的对象是不同的。

除了singlton，Spring还支持prototype、request、session、global-session等按需生成对象的作用域。`BeanFactory.registerScope`方法还支持自定义的作用域。

Bean对象也支持依赖关系，可以从父Bean对象中继承行为。


# 如何定义Bean的供应行为

有`@Configuration`注解标记的的类，表示可以作为Bean对象定义的来源。

`@Import`注解支持Bean的依赖关联。

`@PropertySource`注解支持从外部properties文件中注入属性。

在方法上添加`@Bean`注解表示该方法所返回的对象应该被注册为容器中的一个Bean对象，该对象的id和方法名相同。`@Bean`旁边可以使用`@Scope`注解指定Bean对象的作用域。此外，可以使用`@Primary`注解标记注入时优先调用的Bean。

# 如何定义Bean的加载行为

如果要调用容器中的对象，可以使用`@AutoWired`注解，可以标记在属性上自动匹配适合的bean对象，也可以标记在调用的方法上表示传入的参数通过容器注入。

`@AutoWired`可以支持按属性名、类名来查找合适的Bean对象、如果还没有找到，则会尝试调用构造器，传入容器中已有的合适的参数来生成，如果没有合适的参数，就会报错。如果需要允许`@AutoWired`注解放弃查找，就需要在注解中设置属性`required`为`false`。此外，可以使用`@Qualifier`注解注入指定的Bean对象。

此外，`@Resource`注解可以实现加载指定id的Bean对象。

# Bean的生命周期行为

`@Bean`注解支持通过`init-method`、`destory-method`属性关联无参方法来定义Bean对象的加载后和销毁前的行为。

此外、Spring提供了`InitializingBean`和`DisposableBean`两个接口，可以实现相同的功能。

如果要定义容器的同一行为，可以使用`BeanPostProcessors`和`DestructionAwareBeanPostProcessors`两个接口来实现。`CommonAnnotationBeanPostProcessor`就实现了`@PostConstruct`和`@PreDestroy`注解来实现相同的功能。

`BeanFactory`接口的文档里指出了，Bean对象生命周期内的方法执行顺序。

> The full set of initialization methods and their standard order is:
>
> 1. BeanNameAware -> setBeanName
> 2. BeanClassLoaderAware -> setBeanClassLoader
> 3. BeanFactoryAware -> setBeanFactory
> 4. EnvironmentAware -> setEnvironment
> 5. EmbeddedValueResolverAware -> setEmbeddedValueResolver
> 6. ResourceLoaderAware -> setResourceLoader (only applicable when running in an application context)
> 7. ApplicationEventPublisherAware -> setApplicationEventPublisher (only applicable when running in an application context)
>MessageSourceAware -> setMessageSource (only applicable when running in an application context)
> 8. ApplicationContextAware -> setApplicationContext (only applicable when running in an application context)
> 9. ServletContextAware -> setServletContext (only applicable when running in a web application context)
> 10. postProcessBeforeInitialization methods of BeanPostProcessors
> 11. InitializingBean -> afterPropertiesSet
> 12. a custom init-method definition
> 13. postProcessAfterInitialization methods of BeanPostProcessors
> 
> On shutdown of a bean factory, the following lifecycle methods apply:
> 
> 1. postProcessBeforeDestruction methods of DestructionAwareBeanPostProcessors
> 2. DisposableBean -> destroy
> 3. a custom destroy-method definition

需要注意的是，对于作用域为`prototype`的Bean对象，容器只负责创建不负责销毁，也就是不会触发销毁前的定义方法。


# 文档地址

[Spring Framework - Core Technologies](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html)







