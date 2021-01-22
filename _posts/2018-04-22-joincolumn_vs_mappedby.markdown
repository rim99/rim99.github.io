---
layout: post
title: JoinColumn vs mappedBy
date: 2018-04-22 19:53:02
categories: 原创
---




>Stackoverflow有一道题[JoinColumn vs mappedBy](https://stackoverflow.com/questions/11938253/jpa-joincolumn-vs-mappedby)很有意思：
>
```java
@Entity
public class Company {
    @OneToMany(cascade = CascadeType.ALL , fetch = FetchType.LAZY)
    @JoinColumn(name = "companyIdRef", referencedColumnName = "companyId")
    private List<Branch> branches;
...
}
```
>
```java
@Entity
public class Company {
    @OneToMany(cascade = CascadeType.ALL , fetch = FetchType.LAZY, mappedBy = "companyIdRef")
    private List<Branch> branches;
    ...
}
```
>
>上面两种级联有什么区别？

原题下面有两个高分答案。可惜对于我这种学渣，看完还是一头雾水。

看不懂别人写的，那就自己调调看。

# 准备环境

磨刀不误砍柴工，先准备调试环境。

我用的是Spring Boot 2.0 + Spring Data JPA 2.0， Hibernate版本是5.2.14。

首先在Spring Boot的application.yml里开启Hibernate的调试选项。

```yaml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true  # 打印SQL语句
        format_sql: true  # 格式化SQL语句
        use_sql_comments: true  # 增加注释信息，就知道语句对应的Entity类型了
        generate_statistics: true  # 统计信息，给出了每一步的耗时信息

```

还要在log配置开启`org.hibernate.type`的`debug`级别的日志信息。我用的是log4j2，就需要在log4j2.yml文件中添加:

```yaml
Configuration:
  Loggers:
    Logger:
      - name: org.hibernate.type
        additivity: false
        level: debug  # 这个最关键
        AppenderRef:
          - ref: CONSOLE
          - ref: ROLLING_FILE
```

# 第一次测试

原题里使用的是“公司-部门”模型，我这里本地已经有现成的“部门-雇员”模型，就直接复用了。道理是一样的。

```Java
@Entity
@Table(name="department_demo")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.EAGER, mappedBy = "departmentId")
    private Set<Employee> employeeSet = new HashSet<>();
    
    // setters & getters
}
```

```Java
@Entity
@Table(name="employee_demo")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;

    private Integer departmentId;
    
    // setters & getters
}

```

在这里，Department使用主键`id`一对多的关联于Employee的`departmentId`属性。

然后就是保存一个对象看看。

```Java
@Service
public class DepartmentService {

    @Autowired
    DepartmentRepository departmentRepository;

    @PostConstruct  
    public void insertNewRecord() {

        Department department = new Department();
        department.setName("Leader");
        departmentRepository.save(department);

        Employee emily = new Employee();
        emily.setName("David");

        Employee alice = new Employee();
        alice.setName("Wang Dali");

        department.addEmployee(emily);
        department.addEmployee(alice);

        departmentRepository.save(department);

    }
}
```

启动之后发现log里是这样打印的。

```
2018-04-22 09:11:22,155:INFO restartedMain (StatisticalLoggingSessionEventListener.java:258) - Session Metrics {
    0 nanoseconds spent acquiring 0 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    0 nanoseconds spent preparing 0 JDBC statements;
    0 nanoseconds spent executing 0 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
    0 nanoseconds spent performing 0 L2C puts;
    0 nanoseconds spent performing 0 L2C hits;
    0 nanoseconds spent performing 0 L2C misses;
    0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections);
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)
}
Hibernate: 
    /* insert com.example.demo.model.PO.Department
        */ insert 
        into
            department_demo
            (name) 
        values
            (?)
2018-04-22 09:11:22,314:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [VARCHAR] - [Leader]
Hibernate: 
    select
        currval('department_demo_id_seq')
2018-04-22 09:11:22,438:INFO restartedMain (StatisticalLoggingSessionEventListener.java:258) - Session Metrics {
    733681 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    6711770 nanoseconds spent preparing 2 JDBC statements;
    75097150 nanoseconds spent executing 2 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
    0 nanoseconds spent performing 0 L2C puts;
    0 nanoseconds spent performing 0 L2C hits;
    0 nanoseconds spent performing 0 L2C misses;
    14646304 nanoseconds spent executing 1 flushes (flushing a total of 1 entities and 1 collections);
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)
}
Hibernate: 
    /* load com.example.demo.model.PO.Department */ select
        department0_.id as id1_1_1_,
        department0_.name as name2_1_1_,
        employeese1_.department_id as departme2_2_3_,
        employeese1_.id as id1_2_3_,
        employeese1_.id as id1_2_0_,
        employeese1_.department_id as departme2_2_0_,
        employeese1_.name as name3_2_0_ 
    from
        department_demo department0_ 
    left outer join
        employee_demo employeese1_ 
            on department0_.id=employeese1_.department_id 
    where
        department0_.id=?
2018-04-22 09:11:22,460:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [33]
2018-04-22 09:11:22,499:TRACE restartedMain (BasicExtractor.java:51) - extracted value ([id1_2_0_] : [INTEGER]) - [null]
2018-04-22 09:11:22,502:TRACE restartedMain (BasicExtractor.java:61) - extracted value ([name2_1_1_] : [VARCHAR]) - [Leader]
2018-04-22 09:11:22,503:TRACE restartedMain (BasicExtractor.java:51) - extracted value ([departme2_2_3_] : [INTEGER]) - [null]
Hibernate: 
    /* insert com.example.demo.model.PO.Employee
        */ insert 
        into
            employee_demo
            (department_id, name) 
        values
            (?, ?)
2018-04-22 09:11:22,516:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [33]
2018-04-22 09:11:22,516:TRACE restartedMain (BasicBinder.java:65) - binding parameter [2] as [VARCHAR] - [David]
Hibernate: 
    select
        currval('employee_demo_id_seq')
Hibernate: 
    /* insert com.example.demo.model.PO.Employee
        */ insert 
        into
            employee_demo
            (department_id, name) 
        values
            (?, ?)
2018-04-22 09:11:22,526:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [33]
2018-04-22 09:11:22,527:TRACE restartedMain (BasicBinder.java:65) - binding parameter [2] as [VARCHAR] - [Wang Dali]
Hibernate: 
    select
        currval('employee_demo_id_seq')
```

# 第二次测试

第二次将Department的级联注解做了修改。

```java
@Entity
@Table(name="department_demo")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.EAGER)
    @JoinColumn(name="departmentId", referencedColumnName = "id")
    private Set<Employee> employeeSet = new HashSet<>();
    
    // setters & getters
}    
```

打印的log是这样子。

```
2018-04-22 09:16:29,366:INFO restartedMain (StatisticalLoggingSessionEventListener.java:258) - Session Metrics {
    0 nanoseconds spent acquiring 0 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    0 nanoseconds spent preparing 0 JDBC statements;
    0 nanoseconds spent executing 0 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
    0 nanoseconds spent performing 0 L2C puts;
    0 nanoseconds spent performing 0 L2C hits;
    0 nanoseconds spent performing 0 L2C misses;
    0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections);
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)
}
Hibernate: 
    /* insert com.example.demo.model.PO.Department
        */ insert 
        into
            department_demo
            (name) 
        values
            (?)
2018-04-22 09:16:29,478:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [VARCHAR] - [Leader]
Hibernate: 
    select
        currval('department_demo_id_seq')
2018-04-22 09:16:29,513:INFO restartedMain (StatisticalLoggingSessionEventListener.java:258) - Session Metrics {
    743154 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    4202462 nanoseconds spent preparing 2 JDBC statements;
    9628594 nanoseconds spent executing 2 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
    0 nanoseconds spent performing 0 L2C puts;
    0 nanoseconds spent performing 0 L2C hits;
    0 nanoseconds spent performing 0 L2C misses;
    11708469 nanoseconds spent executing 1 flushes (flushing a total of 1 entities and 1 collections);
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)
}
Hibernate: 
    /* load com.example.demo.model.PO.Department */ select
        department0_.id as id1_1_1_,
        department0_.name as name2_1_1_,
        employeese1_.department_id as departme2_2_3_,
        employeese1_.id as id1_2_3_,
        employeese1_.id as id1_2_0_,
        employeese1_.department_id as departme2_2_0_,
        employeese1_.name as name3_2_0_ 
    from
        department_demo department0_ 
    left outer join
        employee_demo employeese1_ 
            on department0_.id=employeese1_.department_id 
    where
        department0_.id=?
2018-04-22 09:16:29,529:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [34]
2018-04-22 09:16:29,538:TRACE restartedMain (BasicExtractor.java:51) - extracted value ([id1_2_0_] : [INTEGER]) - [null]
2018-04-22 09:16:29,541:TRACE restartedMain (BasicExtractor.java:61) - extracted value ([name2_1_1_] : [VARCHAR]) - [Leader]
2018-04-22 09:16:29,542:TRACE restartedMain (BasicExtractor.java:51) - extracted value ([departme2_2_3_] : [INTEGER]) - [null]
Hibernate: 
    /* insert com.example.demo.model.PO.Employee
        */ insert 
        into
            employee_demo
            (department_id, name) 
        values
            (?, ?)
2018-04-22 09:16:29,552:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [34]
2018-04-22 09:16:29,552:TRACE restartedMain (BasicBinder.java:65) - binding parameter [2] as [VARCHAR] - [Wang Dali]
Hibernate: 
    select
        currval('employee_demo_id_seq')
Hibernate: 
    /* insert com.example.demo.model.PO.Employee
        */ insert 
        into
            employee_demo
            (department_id, name) 
        values
            (?, ?)
2018-04-22 09:16:29,555:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [34]
2018-04-22 09:16:29,556:TRACE restartedMain (BasicBinder.java:65) - binding parameter [2] as [VARCHAR] - [David]
Hibernate: 
    select
        currval('employee_demo_id_seq')
Hibernate: 
    /* create one-to-many row com.example.demo.model.PO.Department.employeeSet */ update
        employee_demo 
    set
        department_id=? 
    where
        id=?
2018-04-22 09:16:29,576:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [34]
2018-04-22 09:16:29,577:TRACE restartedMain (BasicBinder.java:65) - binding parameter [2] as [INTEGER] - [30]
Hibernate: 
    /* create one-to-many row com.example.demo.model.PO.Department.employeeSet */ update
        employee_demo 
    set
        department_id=? 
    where
        id=?
2018-04-22 09:16:29,595:TRACE restartedMain (BasicBinder.java:65) - binding parameter [1] as [INTEGER] - [34]
2018-04-22 09:16:29,596:TRACE restartedMain (BasicBinder.java:65) - binding parameter [2] as [INTEGER] - [29]
2018-04-22 09:16:29,600:INFO restartedMain (StatisticalLoggingSessionEventListener.java:258) - Session Metrics {
    225339 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    731494 nanoseconds spent preparing 7 JDBC statements;
    24985288 nanoseconds spent executing 7 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
    0 nanoseconds spent performing 0 L2C puts;
    0 nanoseconds spent performing 0 L2C hits;
    0 nanoseconds spent performing 0 L2C misses;
    35806114 nanoseconds spent executing 1 flushes (flushing a total of 3 entities and 1 collections);
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)
}
```

# 简单分析

对比log，我们可以看出，两次持久化新的employee对象时，都会：

1. 执行一次left join查询语句，查询department关联的所有employee对象
2. 每一个新的employee对象，都会执行一次插入操作

不同之处在于，**第二次使用`JoinColumn`注解的时候，log显示：每一个新增employee对象都执行了一次update语句，更新了外键**。

原因就在于，`mappedBy`将外键的赋值操作委托给了Employee对象。而`JoinColumn`则选择由Department对象自己来约束外键的关联。

两个注解只有少许区别，但是最终的执行结果差异却很大。多出来的写操作，在生产环境下很容易对数据库构成很大的压力。在代码中完成对Employee对象`departmentId`属性的赋值，显然是一个更为合适的方案。

实际上，上面两个方案是[JPA API文档](https://docs.oracle.com/javaee/7/api/index.html)里`OneToMany`注解的示例2、示例3.

示例1的双向注解，也可以避免多余的写操作。

# 参考资料

1. [[JavaEE - JPA] 性能优化: 如何定位性能问题](https://blog.csdn.net/dm_vincent/article/details/53444490)
2. [The best way to map a @OneToMany relationship with JPA and Hibernate](https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/)
 