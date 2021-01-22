---
layout: post
title: Spring Data JPA的Repository
date: 2018-02-21 19:53:02
categories: 翻译
---

 
> 本文摘译自[官方文档第四章《JPA Repositories》](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.introduction)。版本：2.0.3.RELEASE


# 基本配置

这里是Spring Data JPA的注解风格的配置类示例。（为便于描述，后文直接称Spring Data JPA为框架）。

```Java
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
class ApplicationConfig {

  @Bean
  public DataSource dataSource() {

    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    return builder.setType(EmbeddedDatabaseType.HSQL).build();
  }

  @Bean
  public LocalContainerEntityManagerFactoryBean entityManagerFactory() {

    HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    vendorAdapter.setGenerateDdl(true);

    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setJpaVendorAdapter(vendorAdapter);
    factory.setPackagesToScan("com.acme.domain");
    factory.setDataSource(dataSource());
    return factory;
  }

  @Bean
  public PlatformTransactionManager transactionManager() {

    JpaTransactionManager txManager = new JpaTransactionManager();
    txManager.setEntityManagerFactory(entityManagerFactory());
    return txManager;
  }
}
```

上面这个例子展示了使用Spring的JDBC API - `EmbeddedDatabaseBuilder`设置嵌入式HSQL数据库。然后用Hibernate实现持久化机制。这里使用了`LocalContainerEntityManagerFactoryBean`而不是`EntityManagerFactory`，是因为前者可以更好的处理异常。还有一个基础组件就是`JpaTransactionManager`。最后使用`@EnableJpaRepositories`注解保证每一个注解了`@Repository`的仓储类抛出的异常可以转入到Spring的`DataAccessException`异常体系。如果没有指定基础package，就默认为配置类所在的package。

# 持久化对象

存储持久化对象可以使用`CrudRepository.save`方法。这个方法将持久化对象的持久化(persist)和合并(merge)抽象为一个方法。如果对象还没有持久化，就会调用`entityManager.persist`方法。如果已经持久化，就会调用`entityManager.merge`方法。

## 如何检查实体类的状态

1. 框架默认会检查实体类的主键属性的值，如果为null就表示尚未持久化。
2. 如果实体类实现了`Persistable`接口，框架会调用`isNew`方法。
3. 还可以实现`EntityInformation`接口，但这个方法比较复杂，一般不怎么用，详细请研究文档。


# 查询方法

框架支持函数命名的查询方法定义，也支持注解方式。

函数命名的关键字，可以看[文档](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repository-query-keywords)。

## NamedQuery

`@NamedQuery`注解可以自定义查询语句。这个注解使用在实体类上。

```Java
@Entity
@NamedQuery(name = "User.findByEmailAddress", 
            query = "select u from User u where u.emailAddress = ?1")
public class User {
    ...
}
```

仓储接口的定义。

```Java
public interface UserRepository extends JpaRepository<User, Long> {

  List<User> findByLastname(String lastname);

  User findByEmailAddress(String emailAddress);
}
```

当调用接口方法时，框架首先根据实体类查找是否注解了方法名对应的自定义查询语句。例如，调用`findByEmailAddress`的时候，找到了实体类注解的方法`select u from User u where u.emailAddress = ?1`。

## Query

上面那个方法多少有点不直观。`@Query`注解可以直接在接口方法上注明自定义的查询语句。


```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.emailAddress = ?1")
  User findByEmailAddress(String emailAddress);
}
```

在实际应用中，相比`@NamedQuery`注解，`@Query`注解有更高的优先级。

如果`@Query`注解的`native`值为`true`，方法就可以直接执行SQL语句查询了。

不过，对于这种SQL语句，文档声称目前不支持动态排序查询。对于分页，用于需要指定计数查询语句.

```Java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE LASTNAME = ?1",
    countQuery = "SELECT count(*) FROM USERS WHERE LASTNAME = ?1",
    nativeQuery = true)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

## 排序

`Sort`和`@Query`配合使用比较方便。`Sort`构造器参数必须是查询结果返回的字段，不接受SQL函数。要使用SQL函数，应该用`JpaSort.unsafe`。

```Java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.lastname like ?1%")
  List<User> findByAndSort(String lastname, Sort sort);

  @Query("select u.id, LENGTH(u.firstname) as fn_len from User u where u.lastname like ?1%")
  List<Object[]> findByAsArrayAndSort(String lastname, Sort sort);
}

repo.findByAndSort("lannister", new Sort("firstname"));               // 1    
repo.findByAndSort("stark", new Sort("LENGTH(firstname)"));           // 2
repo.findByAndSort("targaryen", JpaSort.unsafe("LENGTH(firstname)")); // 3
repo.findByAsArrayAndSort("bolton", new Sort("fn_len"));              // 4
```

上面第二个调用是会抛出异常的，应该像第三个方法那样调用。

## 如何使用命名参数

框架默认使用的占位符是按照参数顺序，这样不太直观。使用命名参数，代码能更直观。

```Java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
  User findByLastnameOrFirstname(@Param("lastname") String lastname,
                                 @Param("firstname") String firstname);
}
```

## SpEL表达式

框架还吃支持在`@Query`注解中使用SpEL表达式。

SpEL表达式中可以使用`# {% raw %} {# {% endraw %} entityName}`特指实体类的名称。这个与实体类的`@Entity`注解的`name`属性参数一致。

```Java
@Entity
public class User {

  @Id
  @GeneratedValue
  Long id;

  String lastname;
}

public interface UserRepository extends JpaRepository<User,Long> {

  @Query("select u from #{#entityName} u where u.lastname = ?1")
  List<User> findByLastname(String lastname);
}
```

这种定义方式通常用于定义范型仓储接口。


```Java
@MappedSuperclass
public abstract class AbstractMappedType {
  …
  String attribute
}

@Entity
public class ConcreteType extends AbstractMappedType { … }

@NoRepositoryBean
public interface MappedTypeRepository<T extends AbstractMappedType> extends Repository<T, Long> {

  @Query("select t from #{#entityName} t where t.attribute = ?1")
  List<T> findAllByAttribute(String attribute);
}

public interface ConcreteRepository extends MappedTypeRepository<ConcreteType> { … }
```

## 修改式查询

对于`update`或者`delete`这样的修改式查询，需要在`@Query`注解上增加`@Modifying`注解。执行过查询之后，`EntityManager`有可能会存在过时的实体对象。但是，`EntityManager`默认不会自动更新，因为调用`EntityManager.clear`方法会抹去`EntityManager`所有的未提交修改。如果确认要自动更新，需要将`@Modifying`注解的`clearAutomatically`属性设置为`true`。

框架支持命名式删除语句，也支持注解式。

```Java
interface UserRepository extends Repository<User, Long> {

  void deleteByRoleId(long roleId);

  @Modifying
  @Query("delete from User u where user.role.id = ?1")
  void deleteInBulkByRoleId(long roleId);
}
```

两者在运行时有一个很大的区别。后者仅仅执行JPQL查询，不会触发任何生命周期回调。而前者会在执行完查询之后，调用`CrudRepository.delete(Iterable<User> users)`方法，从而触发`@PreRemove`回调。

## QueryHints

`@QueryHints`注解支持对查询语句进行微调。例如，设置缓存、设置锁超时等等。

可以看看[这篇文章](https://www.thoughts-on-java.org/11-jpa-hibernate-query-hints-every-developer-know/)，讲的不错。

```Java
public interface UserRepository extends Repository<User, Long> {

  @QueryHints(value = { @QueryHint(name = "name", value = "value")},
              forCounting = false)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

`@QueryHints`的`value`项是一组`@QueryHint`，另一个`forCounting`表示是否为可能的聚合查询应用这些微调。例子中，分页查询回去查询总页数，这个子查询不会应用微调。

## 配置加载计划

`@EntityGraph`和`@NamedEntityGraph`配合使用可以实现懒加载多级关联对象。

`@NamedEntityGraph`注解在实体类上，表示的是加载计划。

```Java
@Entity
@NamedEntityGraph(name = "GroupInfo.detail",
  attributeNodes = @NamedAttributeNode("members"))
public class GroupInfo {

  // default fetch mode is lazy.
  @ManyToMany
  List<GroupMember> members = new ArrayList<GroupMember>();

  ...
}
```

`@EntityGraph`表示要执行的加载计划。

```Java
@Repository
public interface GroupRepository extends CrudRepository<GroupInfo, String> {

  @EntityGraph(value = "GroupInfo.detail", type = EntityGraphType.LOAD)
  GroupInfo getByGroupName(String name);

}
```

也可以不用`@NamedEntityGraph`注解，而是直接使用属性`attributePaths`临时设置查询计划。

```Java
@Repository
public interface GroupRepository extends CrudRepository<GroupInfo, String> {

  @EntityGraph(attributePaths = { "members" })
  GroupInfo getByGroupName(String name);

}
```

这个说起来很多内容，具体研究一下JPA 2.1规范的3.7.4章节。


# 存储过程的调用

假设数据库中有这样的存储过程。


```sql
/;
DROP procedure IF EXISTS plus1inout
/;
CREATE procedure plus1inout (IN arg int, OUT res int)
BEGIN ATOMIC
 set res = arg + 1;
END
/;
```


这是一个原子加一的方法。

首先要在实体类上声明过程。


```Java
@Entity
@NamedStoredProcedureQuery(name = "User.plus1", procedureName = "plus1inout", parameters = {
  @StoredProcedureParameter(mode = ParameterMode.IN, name = "arg", type = Integer.class),
  @StoredProcedureParameter(mode = ParameterMode.OUT, name = "res", type = Integer.class) })
public class User {}
```

然后再仓储接口中声明方法。以下四种方式是等效的。


```Java
@Procedure("plus1inout")
Integer explicitlyNamedPlus1inout(Integer arg);
```


```Java
@Procedure(procedureName = "plus1inout")
Integer plus1inout(Integer arg);
```


```Java
@Procedure(name = "User.plus1")
Integer entityAnnotatedCustomNamedProcedurePlus1(@Param("arg") Integer arg);
```


```Java
@Procedure
Integer plus1(@Param("arg") Integer arg);
```


# `Specification`


JPA 2.0 引入了`criteria` API能够以代码的方式构建查询。`criteria` API其实就是为领域类的查询操作构建where子句。退一步来看，其实`criteria`也就是一种谓词(predicate)。Spring Data JPA框架接受了Eric Evans的《Domain Driven Design》一书的Specification概念，拥有与`criteria`相似的API。

首先，仓储接口必须继承`JpaSpecificationExecutor`接口。

```Java
public interface CustomerRepository extends CrudRepository<Customer, Long>, JpaSpecificationExecutor {
 …
}
```

该接口定义了一系列方法，可以实现谓词的可变性。

```Java
List<T> findAll(Specification<T> spec);
```

实际上，`Specification`也是一个接口。

```Java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder builder);
}
```

`Specification`可以很方便的构建新谓词。看个例子。

先定义基础的`Specification`。

```Java
public class CustomerSpecs {

  public static Specification<Customer> isLongTermCustomer() {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<Customer> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         LocalDate date = new LocalDate().minusYears(2);
         return builder.lessThan(root.get(_Customer.createdAt), date);
      }
    };
  }

  public static Specification<Customer> hasSalesOfMoreThan(MontaryAmount value) {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         // build query here
      }
    };
  }
}
```

这时使用方法。


```Java
List<Customer> customers = customerRepository.findAll(isLongTermCustomer());
```

这样可以构建新的复杂谓词。


```Java
MonetaryAmount amount = new MonetaryAmount(200.0, Currencies.DOLLAR);
List<Customer> customers = customerRepository.findAll(
                               where(isLongTermCustomer()).or(hasSalesOfMoreThan(amount)));
```

# 事务

仓储接口对象的CRUD方法均默认具备事务性。读取查询的`readonly`属性默认为`true`。具体可看文档[SimpleJpaRepository](https://docs.spring.io/spring-data/data-jpa/docs/current/api/index.html?org/springframework/data/jpa/repository/support/SimpleJpaRepository.html)。要想修改事务配置，需要覆盖原来的方法。

```Java
public interface UserRepository extends CrudRepository<User, Long> {

  @Override
  @Transactional(timeout = 10)
  public List<User> findAll();

  // Further query method declarations
}
```

上面这个例子设置了10s超时。

还有一种方法是在service层进行调整。

```Java
@Service
class UserManagementImpl implements UserManagement {

  private final UserRepository userRepository;
  private final RoleRepository roleRepository;

  @Autowired
  public UserManagementImpl(UserRepository userRepository,
    RoleRepository roleRepository) {
    this.userRepository = userRepository;
    this.roleRepository = roleRepository;
  }

  @Transactional
  public void addRoleToAllUsers(String roleName) {

    Role role = roleRepository.findByName(roleName);

    for (User user : userRepository.findAll()) {
      user.addRole(role);
      userRepository.save(user);
    }
}
```

上面这个例子实现了`addRoleToAllUsers`方法的事务性，而方法内部调用的事务性会被忽视。如果想要在facade里面配置事务性，需要增加注解`@EnableTransactionManagement`。

接口定义处也可以注解`@Transactional`，但是优先级低于方法定义处的同类注解。

# 锁

框架支持为查询操作加锁。

```Java
interface UserRepository extends Repository<User, Long> {

  // Plain query method
  @Lock(LockModeType.READ)
  List<User> findByLastname(String lastname);
}
```



