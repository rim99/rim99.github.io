---
layout: post
title: Spring Data 文档摘译
date: 2018-02-18 19:53:02
categories: 翻译
---


# 如何实现数据库管理操作

看官方文档给的例子。

首先定义一个自有的repository接口，继承自Repository接口或者其衍生接口。定义时，注明领域类型和主键类型。然后在接口内声明方法。然后要在配置类上标注`@EnableJpaRepositories`，让Spring容器自行创建接口的实例。

```Java
/* 
 * 领域类为User， 主键类型是Long
 */
interface PersonRepository extends Repository<User, Long> {
	List<Person> findByLastname(String lastname);
}

```
```Java
@EnableJpaRepositories
public class Config() {}
```

最后，在需要实现操作的地方直接注入自定义的repository接口就好了。

```Java
public class SomeClient {

  @Autowired
  private PersonRepository repository;

  public void doSomething() {
    List<Person> persons = repository.findByLastname("Matthews");
  }
}

```

# Repository

Repository是Spring Data的核心概念，负责管理领域类(domain class)。

CrudRepository是最基本的Repository接口，实现了增删查改功能。PagingAndSortingReporitory进行了拓展，可以实现结果列表的排序和分页。

上面的例子中使用了`@EnableJpaRepositories`注解生成自定义repository接口的实例，如果需要在Spring容器外调用，可以调用工厂生成实例。

```Java
RepositoryFactorySupport factory = …    // 初始化工厂
UserRepository repository = factory.getRepository(UserRepository.class);
```

如果想要拓展Spring Data定义好的方法，需要写一个对应接口的实现类。在实现类中用定义自己的方法。实现类名一定要**以Impl结尾**。

如果覆写的是基础Repository接口方法，需要在`@EnableJpaRepositories`的`repositoryBaseClass`属性指定对应的实现类。

如果某些Repository接口不需要在运行时生成对象，可以在接口定义加上注解`@NoRepositoryBean`。

## 空指针Null

Spring Data的crud方法支持返回Java 8的`Optional`对象，暗示了结果有可能为Null。

此外，Spring Data还支持三类注解：

1. `@NonNullApi`: package级别的注解，表示其下所有方法的参数和返回值都不会是Null
2. `@NonNull`: 方法级或参数级注解，前者表示返回值非Null，后者表示参数非Null
3. `@Nullable`: 和上一个注解相反，表示返回值或参数可以为Null

如果使用了第一个注解，就没有必要使用第二个注解。

## 对领域事件的支持

在DDD领域驱动设计模式中，和Repository关联的实体类都是聚合根(aggregate root)。聚合根会发布领域事件。Spring Data对此提供了`@DomainEvents`注解方便了时间的发布。

```Java
class AnAggregateRoot {

    @DomainEvents 
    Collection<Object> domainEvents() {
        // … return events you want to get published here
    }

    @AfterDomainEventPublication 
    void callbackMethod() {
       // … potentially clean up domain events list
    }
}
```

1. `@DomainEvents`注解的方法可以返回单个事件，也可以返回多个事件的集合。这个方法不能有入参。
2. 所有事件发布后，`@AfterDomainEventPublication`注解的方法可以清理要发布的事件列表。

## QueryDslPredicateExecutor

继承了`QueryDslPredicateExecutor`接口的repository接口，可以实现查找满足自定义`Predicate`对象的结果。

定义

```Java
interface UserRepository extends CrudRepository<User, Long>, QueryDslPredicateExecutor<User> {

}
```

调用

```Java
Predicate predicate = user.firstname.equalsIgnoreCase("dave")
	.and(user.lastname.startsWithIgnoreCase("mathews"));

userRepository.findAll(predicate);
```

# 映射

Spring Data的find方法是可以返回整行数据的。但是通常，程序只需要其中部分字段的数据。对于这种情况，可以使用自定义的接口来完成。

下面是一个简单的repository接口和领域类例子。

```Java
class Person {

  @Id UUID id;
  String firstname, lastname;
  Address address;

  static class Address {
    String zipCode, city, street;
  }
}

interface PersonRepository extends Repository<Person, UUID> {

  Collection<Person> findByLastname(String lastname);
}
```

如果我们只对其中的`firstname`和`lastname`感兴趣，我们可以写一个新的接口。注意：这接口的getter必须是领域类属性的子集。

```Java
interface NamesOnly {

  String getFirstname();
  String getLastname();
}
```
 
然后，将repository接口里的方法改成这样。

```Java
interface PersonRepository extends Repository<Person, UUID> {

  Collection<NamesOnly> findByLastname(String lastname);
}

```

查询引擎会在运行时，为每一个查询结果创建接口的代理对象。

映射接口的支持递归定义。

```Java
interface PersonSummary {

  String getFirstname();
  String getLastname();
  AddressSummary getAddress();

  interface AddressSummary {
    String getCity();
  }
}
```


## 闭合映射

闭合映射是指接口的getter总是领域类属性的子集（含全集）。

Spring Data可以优化使用闭合映射的场景，因为暴露给调用方的属性都是定义的类型。

## 开放映射

接口的getter可以使用`@Value`注解，来计算新值。

```Java
interface NamesOnly {

  @Value("#{target.firstname + ' ' + target.lastname}")
  String getFullName();
  …
}
```

在上面这个例子中，注解里面是SpEL表达式。`target`表示的是整个领域类。这种getter对应的属性在领域类中并不存在，所以称作开放映射。`@Value`注解里的表达式不能太复杂，毕竟结果是String类型。

对于简单的表达式，也可以使用`default`方法。但这要求方法的逻辑只能依赖于映射接口实现的其他getter方法。

```Java
interface NamesOnly {

  String getFirstname();
  String getLastname();

  default String getFullName() {
    return getFirstname.concat(" ").concat(getLastname());
  }
}

```

接口方法中的参数也可以在表达式中调用。

```Java
interface NamesOnly {

  @Value("#{args[0] + ' ' + target.firstname + '!'}")
  String getSalutation(String prefix);
}
```

还有一种更灵活的方法。将定制逻辑在其他Spring Bean上实现出来，然后在SpEL表达式中实现出来。看个例子。

```Java
@Component
class MyBean {

  String getFullName(Person person) {
    …
  }
}

interface NamesOnly {

  @Value("#{@myBean.getFullName(target)}")
  String getFullName();
  …
}
```

## 映射类

Spring Data也支持DTO类型的实体类来映射。DTO类属性为领域类属性的子集。

当然，使用DTO的时候，运行时不会创建代理对象。

Spring Data在优化查询字段的时候，只会选择构造器的参数。(*这个听上去比接口靠谱哇*)

```Java
class NamesOnly {

  private final String firstname, lastname;

  NamesOnly(String firstname, String lastname) {

    this.firstname = firstname;
    this.lastname = lastname;
  }

  String getFirstname() {
    return this.firstname;
  }

  String getLastname() {
    return this.lastname;
  }

  // equals(…) and hashCode() implementations
}
```

## 动态映射

如果想要在调用的时候再决定返回类型，可以这样做。

```Java
interface PersonRepository extends Repository<Person, UUID> {

  Collection<T> findByLastname(String lastname, Class<T> type);
}

/*
 *  动态的传入返回类型
 */
void someMethod(PersonRepository people) {

  Collection<Person> aggregates =
    people.findByLastname("Matthews", Person.class);

  Collection<NamesOnly> aggregates =
    people.findByLastname("Matthews", NamesOnly.class);
}

```


# `Example`查询

`Example`查询可以动态的创建查询，而不需要指定所查询的字段。

首先，得有一个领域类。

```Java
public class Person {

  @Id
  private String id;
  private String firstname;
  private String lastname;
  private Address address;

  // … getters and setters omitted
}
```

有了领域类对象，就可以创建`Example`对象了。默认的，值为Null的属性会被忽视掉。`Example`对象可以使用`of`工厂方法来创建，也可以用`ExampleMatcher`来创建。

```Java
Person person = new Person();                         
person.setFirstname("Dave");                          
Example<Person> example = Example.of(person);
```

调用需要继承`QueryByExampleExecutor`接口。

```Java
public interface QueryByExampleExecutor<T> {

  <S extends T> S findOne(Example<S> example);

  <S extends T> Iterable<S> findAll(Example<S> example);

  // … more functionality omitted.
}

```

调用接口方法，就可以查询满足指定条件的结果。

`ExampleMatcher`可以实现更多条件的String类型的查询。


# 审计

Spring Data 可以通过注解或者接口实现复杂的审计功能，获知谁在何时创建了（或修改了）一条数据。

## 注解实现

注解有`@CreatedBy`、`@LastModifiedBy`、`@CreatedDate`、`@LastModifiedDate`可选。时间可以是`Date`类型，也可以是`Datetime`类型，也支持`long`类型。

```Java
class Customer {

  @CreatedBy
  private User user;

  @CreatedDate
  private DateTime createdDate;

  // … further properties omitted
}
```

## 接口实现

可以考虑让领域类实现`Auditable`接口，也可以继承`AbstractAuditable`抽象类。

文档更推荐注解实现，因为这样更直观，也更灵活。

## `AuditorAware`

提供了可以让服务获知当前用户信息的接口。













