---
title: 深入浅出 Spring Data JPA
tags:
  - 数据库
  - Spring Boot
  - Spring Data JPA
categories:
  - 技术
copyright: true
date: 2020-07-10 17:31:44
---

Spring Data JPA 是在 JPA 规范的基础上进行进一步封装的产物，和之前的 JDBC、slf4j 这些一样，只定义了一系列的接口。具体在使用的过程中，一般接入的是 Hibernate 的实现，那么具体的 Spring Data JPA 可以看做是一个面向对象的 ORM。虽然后端实现是 Hibernate，但是实际配置和使用比 Hibernate 简单不少，可以快速上手。如果业务不太复杂，个人觉得是要比 Mybatis 更简单好用。

本文就由浅入深结合笔者在使用上的经验讲讲 Spring Data JPA 的用法及相关的知识点。

<!-- more -->

## 基本使用

1. **创建项目，选择相应的依赖。一般不直接用 mysql 驱动，而选择连接池。**

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
   <groupId>mysql</groupId>
   <artifactId>mysql-connector-java</artifactId>
   <scope>runtime</scope>
</dependency>
<dependency>
   <groupId>com.alibaba</groupId>
   <artifactId>druid-spring-boot-starter</artifactId>
   <version>1.1.18</version>
</dependency>
```

2. **配置全局 yml 文件。**

```yaml
spring:
 datasource:
   type: com.alibaba.druid.pool.DruidDataSource
   driver-class-name: com.mysql.cj.jdbc.Driver
   url: jdbc:mysql://172.21.30.61:3306/gpucluster?serverTimezone=Hongkong&characterEncoding=utf-8&useSSL=false
   username:
   password:
 jpa:
    hibernate:
      ddl-auto: update
    open-in-view: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL57Dialect
        show_sql: false
        format_sql: true
logging:
  level:
    root: info # 是否需要开启 sql 参数日志
    org.springframework.orm.jpa: DEBUG
    org.springframework.transaction: DEBUG
    org.hibernate.engine.QueryParameters: debug
    org.hibernate.engine.query.HQLQueryPlan: debug
    org.hibernate.type.descriptor.sql.BasicBinder: trace
```

- `hibernate.ddl-auto: update ` 实体类中的修改会同步到数据库表结构中，慎用。
- `show_sql` 可开启 hibernate 生成的 sql，方便调试。
- `logging` 下的几个参数用于显示 sql 的参数。

3. **创建实体类并添加 JPA 注解**

```java
@Entity
@Table(name = "user")
@Data
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private Integer age;
    private String address;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}
```
4. **创建对应的 Repository**

实现 JpaRepository 接口，生成基本的 CRUD 操作样板代码。并且可根据 Spring Data JPA 自带的 Query Lookup Strategies 创建简单的查询操作，在 IDEA 中输入 `findBy ` 等会有提示。

```java
public interface IUserRepository extends JpaRepository<User,Long> {
    List<User> findByName(String name);
    List<User> findByAgeAndCreateTimeBetween(Integer age, LocalDateTime createTime, LocalDateTime createTime2);
}
```
## 查询

### 默认方法

Repository 继承了 `JpaRepository` 后会有一系列基本的默认 CRUD 方法，例如：

```java
List<T> findAll();
Page<T> findAll(Pageable pageable);
T getOne(ID id);
T S save(T entity);
void deleteById(ID id);
```

### 声明式查询

Repository 继承了 `JpaRepository` 后，可在接口中定义一系列方法，它们一般以 `findBy`、`countBy`、`deleteBy`、`existsBy` 等开头，如果使用 IDEA，输入以下关键字后会有响应的提示。例如：

```java
public interface IUserRepository extends JpaRepository<User,Integer>{
    User findByUsername(String username);
    Integer countByDept(String dept);
}
```

对于一些单表多字段查询，使用这种方式就非常舒服了，而且完全 oop 思想，不需要思考具体的 SQL 怎么写。但有个问题，字段多了之后方法名会很长，调用的时候会比较难受，这个时候可以利用 jdk8 的特性将它缩短，当然这种情况也可以直接用 `@Query` 写 HQL 或 SQL 解决。

```java
User findFirstByEmailContainsIgnoreCaseAndField1NotNullAndField2NotNull(final String email);

default User getByEmail(final String email) {
	return findFirstByEmailContainsIgnoreCaseAndField1NotNullAndField2NotNull(email);
}
```

> 常见的操作可见 [附录 - 支持的方法关键词](### 支持的方法关键词)

### 使用注解和 SQL

```java
@Transactional(readOnly = true)
public interface UserRepository extends JpaRepository<User, Long> {
	@Query(nativeQuery = true, value = "select * from user where tel = ?1")
	List<User> getUser(String tel);

	@Modifying
	@Transactional
	@Query("delete from User u where u.active = false")
	void deleteInactiveUsers();
}
```

1. `@Query` 中可写 HQL 和 SQL，如果是 SQL，则 `nativeQuery = true`。

### 复杂查询 Specification

```java
// 复杂查询，创建 Specification
private Page<OrderInfoEntity> getOrderInfoListByConditions(String tel, int pageSize, int pageNo, String beginTime, String endTime) {
    Specification<OrderInfoEntity> specification = new Specification<OrderInfoEntity>() {
        @Override
        public Predicate toPredicate(Root<OrderInfoEntity> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
            List<Predicate> predicate = new ArrayList<>();
            if (!Strings.isNullOrEmpty(beginTime)) {
                predicate.add(cb.greaterThanOrEqualTo(root.get("createTime"), DateUtils.getDateFromTimestamp(beginTime)));
            }
            if (!Strings.isNullOrEmpty(endTime)) {
                predicate.add(cb.lessThanOrEqualTo(root.get("createTime"), DateUtils.getDateFromTimestamp(endTime)));
            }
            if (!Strings.isNullOrEmpty(tel)) {
                predicate.add(cb.equal(root.get("userTel"), tel));
            }
            return cb.and(predicate.toArray(new Predicate[predicate.size()]));
        }
    };
    Sort sort = new Sort(Sort.Direction.DESC, "createTime");
    Pageable pageable = new PageRequest(pageNo - 1, pageSize, sort);
    return orderInfoRepository.findAllEntities(specification, pageable);
}
```

### 子查询

    Specification<UserProject> specification = (root, criteriaQuery, criteriaBuilder) -> {
    	Subquery subQuery = criteriaQuery.subquery(String.class);
    	Root from = subQuery.from(User.class);
    	subQuery.select(from.get("userId")).where(criteriaBuilder.equal(from.get("username"), "mqy6289"));
    	return criteriaBuilder.and(root.get("userId").in(subQuery));
    };
    return userProjectRepository.findAll(specification);

## 删除和修改

- 删除

1. 直接使用默认的 `deleteById()`。
2. 使用申明式查询创建对应的删除方法 `deleteByXxx`。
3. 使用 SQL\HQL 注解删除


- 新增和修改

调用 save 方法，如果是修改的需要先查出相应的对象，再修改相应的属性。

## 事务

Spring Boot 默认集成事务，所以无须手动开启使用 `@EnableTransactionManagement` 注解，就可以用 `@Transactional` 注解进行事务管理。需要使用时，可以查具体的参数。

`@Transactional` 注解的使用，具体可参考：[透彻的掌握 Spring 中 @transactional 的使用](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html)。

谈谈几点用法上的总结：

1. 持久层方法上继承 `JpaRepository`，对应实现类 `SimpleJpaRepository` 中包含 `@Transactional(readOnly = true)` 注解，因此**默认持久层中的 CRUD 方法均添加了事务**。
2. 申明式事务更常用的是在 service 层中的方法上，一般会调用多个 Repository 来完成一项业务逻辑，过程中可能会对多张数据表进行操作，出现异常一般需要级联回滚。一般操作，直接在 Serivce 层方法中添加 `@Transactional` 即可，默认使用数据的隔离级别，默认所有 Repository 方法加入 Service 层中的事务。
3. `@Transactional` 注解中最核心的两个参数是 `propagation` 和 `isolation`。前者用于控制事务的传播行为，指定小事务加入大事务还是所有事务均单独运行等；后者用于控制事务的隔离级别，默认和 MySQL 保持一致，为不可重复读。我们也可以通过这个字段手动修改单个事务的隔离级别。具体的应用场景可见我另一篇博客 [谈谈事务的隔离性及在开发中的应用](https://furur.xyz/2020/06/10/talk-about-transaction-isolation-in-springboot.html)。
4. 同一个 service 层中的方法调用，**如果添加了 `@Transactional` 会启动 hibernate 的一级缓存**，相同的查询多次执行会进行 Session 层的缓存，否则，多次相同的查询作为事务独立执行，则无法缓存。
5. 如果你使用了关系注解，在懒加载的过程中一般都会遇到过 `LazyInitializationException` 这个问题，可通过添加 `@Transactional`，将 session 托管给 Spring Transaction 解决。
6. 只读事务的使用。可在 service 层中全局配置只读事务 `@Transactional(readOnly =true)`，对于具有读写的事务可在对应方法中覆盖即可。在只读事务无法进行写入操作，这样在事务提交前，hibernate 就会跳过 dirty check，并且 Spring 和 JDBC 会有多种的优化，使得查询更有效率。

## JPA Audit
JPA 自带的 Audit 可以通过 AOP 的形式注入，在持久化操作的过程中添加创建和更新的时间等信息。具体使用方法：


1. 申明实体类，需要在类上加上注解 @EntityListeners(AuditingEntityListener.class)。
2. 在 Application 启动类中加上注解 @EnableJpaAuditing
3. 在需要的字段上加上 @CreatedDate、@CreatedBy、@LastModifiedDate、@LastModifiedBy 等注解。

**如果只需要更新创建和更新的时间是不需要额外的配置的。**

## 数据库关系

如果需要进行级联查询，可用  JPA  的 @OneToMany、@ManyToMany 和 @OneToOne 来修饰，当然，碰到出现一对多等情况的时候，可以手动将多的一方的数据去查询出来填充进去。

由于数据库设计的不同，注解在使用上也会存在不同。这里举一个 OneToMany 的例子。

仓库和货物是一对多关系，并且在设计上，Goods 表中包含 Repository 的外键，则在 Repository 添加注解，Goods 上不需要。

```java
@Entity
public class Repository{
  @OneToMany(cascade = {CascadeType.ALL})
  @JoinColumn(name = "repo_id")
  private List<Goods> list;
}

public class Goods{
}
```

具体可参考：[@OneToMany、@ManyToOne 以及 @ManyToMany 讲解（五）](https://my.oschina.net/liangbo/blog/92301)

JPA 的这几个注解和 Hibernate 的关联度比较大，而且一般适合于 code first 的形式，也就是说先有实体类后生成数据库。在这里我并不建议没有学习过 Hibernate 直接上手 Spring Data JPA 的人去使用这些注解，因为一旦加上关系注解后，从查询的角度虽然方便了，但是涉及到一些级联的操作，例如删除、修改等操作，容易采坑。需要额外去了解 Hibernate 的缓存刷新机制。

## 多数据源

默认单数据源的情况下，我们只需要将自己的 Repository 实现 JpaRepository 接口即可，通过 Spring Boot 的 Auto Configuration 会自动帮我们注入所需的 Bean，例如 `LocalContainerEntityManagerFactoryBean`、`EntityManager `、`DataSource`。

但是在多数据源的情况下，就需要根据配置文件去条件化创建这些 Bean 了。

1. **配置文件添加多个数据源信息**

```yaml
spring:
  datasource:
    hangzhou: # datasource1
      type: com.alibaba.druid.pool.DruidDataSource
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://172.17.11.72:3306/gpucluster?serverTimezone=Hongkong&characterEncoding=utf-8&useSSL=false
      username: root
      password: 123456
    shanghai: # datasource2
      type: com.alibaba.druid.pool.DruidDataSource
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://172.21.30.61:3306/gpucluster?serverTimezone=Hongkong&characterEncoding=utf-8&useSSL=false
      username: root
      password: 123456
  jpa:
    open-in-view: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL57Dialect
```

2. **数据源 bean 注入**

```java
@Slf4j
@Configuration
public class DataSourceConfiguration {
    @Bean(name = "HZDataSource")
    @Primary
    @Qualifier("HZDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.hangzhou")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().type(DruidDataSource.class).build();
    }

    @Bean(name = "SHDataSource")
    @Qualifier("SHDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.shanghai")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().type(DruidDataSource.class).build();
    }
}
```

3. **注入 JPA 相关的 bean（一个数据源一个配置文件）**

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef = "entityManagerFactoryHZ",
        transactionManagerRef = "transactionManagerHZ",
        basePackages = {"cn.com.arcsoft.app.repo.jpa.hz"},
        repositoryBaseClass = IBaseRepositoryImpl.class)
public class RepositoryHZConfig {
    private final DataSource HZDataSource;
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public RepositoryHZConfig(@Qualifier("HZDataSource") DataSource HZDataSource, JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.HZDataSource = HZDataSource;
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Primary
    @Bean(name = "entityManagerFactoryHZ")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryHZ(EntityManagerFactoryBuilder builder) {
        // springboot 2.x
        Map<String, Object> properties = hibernateProperties.determineHibernateProperties(
                jpaProperties.getProperties(), new HibernateSettings());
        return builder.dataSource(HZDataSource)
                .properties(properties)
                .packages("cn.com.arcsoft.app.entity")
                .persistenceUnit("HZPersistenceUnit")
                .build();
    }

    @Primary
    @Bean(name = "transactionManagerHZ")
    public PlatformTransactionManager transactionManagerHZ(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryHZ(builder).getObject());
    }
}
```

4. **在之前配置的对应的包中添加相应的 repository 就可以了。如果数据源数据库是相同的，可实现一个主的 repository，其余继承一下。**

```java
@Primary
@Qualifier("volumeHZRepository")
public interface IVolumeRepository extends IBaseRepository<Volume, Integer> {
    Volume findByUserIdAndIp(Integer userId, String ip);
}

@Qualifier("volumeSHRepository")
public interface IVolumeSHRepository extends IVolumeRepository {
}
```

## JPA 与 Hibernate

在使用 Spring Data JPA 的时候，虽然底层是 Hibernate 实现的，但是我们在使用的过程中完全没有感觉，因为我们在使用 JPA 规范提供的 API 来操作数据库。但是遇到一些复杂的业务，或许任然需要关注 Hibernate，或者 JPA 底层的一些实现，例如 EntityManager 和 EntityManagerFactory 的创建和使用。

下面我就讲讲最核心的两点。

### 对象生命周期

用过 Mybatis 的都知道，它属于半自动的 ORM，仅仅是将 SQL 执行后的结果映射到具体的对象，虽然它也做了对查询结果的缓存，但是一旦数据查出来封装到实体类后，就和数据库无关了。但是 JPA 后端的 Hibernate 则不同，作为全自动的 ORM，它自己有一套比较复杂的机制，用于处理对象和数据库中的关系，两者直接会进行绑定。

首先在 Hibernate 中，对象就不再是基本的 Java POJO 了，而是有四种状态。

> 1. 临时状态 (transient): 刚用 new 语句创建，还未被持久化的并且不在 Session 的缓存中的实体类。
> 2. 持久化状态 (persistent): 已被持久化，并且在 Session 缓存中的实体类。
> 3. 删除状态 (removed): 不在 Session 缓存中，而且 Session 已计划将其从数据库中删除的实体类。
> 4. 游离状态 (detached): 已被持久化，但不再处于 Session 的缓存中的实体类。

![image002-35](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/simple-way-to-learn-spring-data-jpa/image002-35.png?x-oss-process=style/logo)

**需要特别关注的是持久化状态的对象，这类对象一般是从数据库中查询出来的，同时会存在 Session 缓存中，由于存在缓存清理与 dirty checking 机制，当修改了对象的属性，无需手动执行 save 方法，当事务提高后，改动会自动提交到数据库中去。**

### 缓存清理与 dirty checking

当事务提交后，会进行缓存清理操作，所有 session 中的持久化对象都会进行 dirty checking。简单描述一下过程：

1. 在一个事务中的各种查询结果都会缓存在对应的 session 中，并且存一份快照。
2. 在事务 commit 前，会调用 `session.flush()` 进行缓存清理和 dirty checking。将所有 session 中的对象和对应快照进行对比，如果发生了变化，则说明该对象 dirty。
3. 执行 update 和 delete 等操作将 session 中变化的数据同步到数据库中。

开启只读事务可屏蔽 dirty checking，提高查询效率。

## Troubleshooting

1. **Jpa 与 lombok 配合使用的问题产生 StackOverflowError**

使用 Hibernate 的关系注解 @ManyToMany 时使用 @Data，执行查询时会出现 StackOverflowError 异常。主要是因为 @Data 帮我们实现了 hashCode() 方法出现了问题，出现了循环依赖。

解决方法：在关系字段上加上 @EqualsAndHashCode.Exclude 即可。

```java
@EqualsAndHashCode.Exclude
@ManyToMany(fetch = FetchType.LAZY,cascade = {CascadeType.PERSIST})
private Set<User> membersSet;
```

[Lombok.hashCode issue with “java.lang.StackOverflowError: null”](https://github.com/rzwitserloot/lombok/issues/1007)

2. **Spring boot JPA:Unknown entity 解决方法**

在采用两个大括号初始化对象后，再调用 JPA 的 save 方法时会抛出 Unknown entity 这个异常，这是 JPA 无法正确识别匿名内部类导致的。

解决方法：手动 new 一个对象再调用 set 方法赋值。

[Spring boot JPA:Unknown entity 解决方法](https://www.jianshu.com/p/179bdaab348d)

3. **使用关系注解时产生的 LazyInitializationException 异常**

> org.hibernate.LazyInitializationException: could not initialize proxy - no Session

如果使用 Hibernate 关系注解，可能会遇到这个问题，这是因为在 session 关闭后 get 对象中懒加载的值产生的。

解决方法：

1. 在实体类中添加注解 `@Proxy(lazy = false)`
2. 在 services 层的方法中添加 `@Transactional`，将 session 管理交给 spring 事务

## 总结

本文主要讲了下 Spring Data JPA 的基本使用和一些个人经验。

ORM 发展至今，从 Hibernate 到 JPA，再到现在的 Spring Data JPA。可以看到是一个不断简化的过程，过去大段的 xml 已经没有了，仅保留基本的 sql 字符串即可。Spring Data JPA 虽然配置和使用起来简单，但由于它的底层依然是 Hibernate 实现的，因此有些东西仍然需要去了解。就目前使用而言，有以下几点感受：

1. 要用好 Spring Data JPA，Hibernate 的相关机制还是需要有一定的了解的，例如前面提到的对象声明周期及 Session 刷新机制等。如果不了解，容易出现一些莫名其妙的问题。
2. 如果是新手，个人不推荐使用关系注解。 技术本身就是一步步在简化，如果不是非常复杂的例如 ERP 系统，没必要去使用 JPA 和 Hibernate 原生的东西，完全可以手动多次查询操作来代替关系注解。之所以这么讲，是因为对 JPA 的关系注解的使用，以及各种级联操作的类型理解不深，会存在一些隐患。


## 参考文献

- [SpringBoot 整合 Spring-data-jpa](https://chenjiabing666.github.io/2018/12/20/SpringBoot%E6%95%B4%E5%90%88Spring-data-jpa/)
- [Spring Boot(五)：Spring Boot Jpa 的使用](http://www.ityouknow.com/springboot/2016/08/20/spring-boot-jpa.html)
- [Spring Data JPA 进阶（六）：事务和锁](https://www.yasinshaw.com/articles/39)
- [Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/)

## 附录

### 支持的方法关键词

| Keyword           | Sample                                                    | JPQL snippet                                       |
| ----------------- | --------------------------------------------------------- | -------------------------------------------------- |
| And               | findByLastnameAndFirstname                                | … where x\.lastname = ?1 and x\.firstname = ?2     |
| Or                | findByLastnameOrFirstname                                 | … where x\.lastname = ?1 or x\.firstname = ?2      |
| Is,Equals         | findByFirstname，findByFirstnameIs，findByFirstnameEquals | … where x\.firstname = ?1                          |
| Between           | findByStartDateBetween                                    | … where x\.startDate between ?1 and ?2             |
| LessThan          | findByAgeLessThan                                         | … where x\.age < ?1                                |
| LessThanEqual     | findByAgeLessThanEqual                                    | … where x\.age <= ?1                               |
| GreaterThan       | findByAgeGreaterThan                                      | … where x\.age > ?1                                |
| GreaterThanEqual  | findByAgeGreaterThanEqual                                 | … where x\.age >= ?1                               |
| After             | findByStartDateAfter                                      | … where x\.startDate > ?1                          |
| Before            | findByStartDateBefore                                     | … where x\.startDate < ?1                          |
| IsNull            | findByAgeIsNull                                           | … where x\.age is null                             |
| IsNotNull,NotNull | findByAge\(Is\)NotNull                                    | … where x\.age not null                            |
| Like              | findByFirstnameLike                                       | … where x\.firstname like ?1                       |
| NotLike           | findByFirstnameNotLike                                    | … where x\.firstname not like ?1                   |
| StartingWith      | findByFirstnameStartingWith                               | … where x\.firstname like ?1（附加参数绑定 %）     |
| EndingWith        | findByFirstnameEndingWith                                 | … where x\.firstname like ?1（与前置绑定的参数 %） |
| Containing        | findByFirstnameContaining                                 | … where x\.firstname like ?1（参数绑定包装 %）     |
| OrderBy           | findByAgeOrderByLastnameDesc                              | … where x\.age = ?1 order by x\.lastname desc      |
| Not               | findByLastnameNot                                         | … where x\.lastname <> ?1                          |
| In                | findByAgeIn\(Collection<Age> ages\)                       | … where x\.age in ?1                               |
| NotIn             | findByAgeNotIn\(Collection<Age> ages\)                    | … where x\.age not in ?1                           |
| True              | findByActiveTrue\(\)                                      | … where x\.active = true                           |
| False             | findByActiveFalse\(\)                                     | … where x\.active = false                          |
| IgnoreCase        | findByFirstnameIgnoreCase                                 | … where UPPER\(x\.firstame\) = UPPER\(?1\)         |
| Top 或者 First    | findTopByNameAndAge，findFirstByNameAndAge                | where … limit 1                                    |
| Topn 或者 Firstn  | findTop2ByNameAndAge，findFirst2ByNameAndAge              | where … limit 2                                    |
| Distinct          | findDistinctPeopleByLastnameOrFirstname                   | select distinct …\.                                |
| count             | countByAge，count                                         | select count\(\*\)                                 |