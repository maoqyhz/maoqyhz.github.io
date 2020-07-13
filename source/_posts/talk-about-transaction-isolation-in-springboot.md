---
title: 谈谈事务的隔离性及在开发中的应用
tags:
  - 事务
  - 数据库
  - Spring Boot
categories:
  - 技术
copyright: true
date: 2020-06-10 19:12:14
---

对于关系型数据库事务，之前的理解还比较浅显，基本还停留在面试宝典中长期背诵的那些以及最基本的操作上，比如一个事务可以执行一对 SQL，一旦遇到异常后会全部回滚，不会造成脏数据。这里体现的是事务的原子性、一致性和持久性。对于隔离性，在之前的开发中基本没有用到过，一直用的数据库默认的隔离级别，也就没用更新的认识了。直到最近开发中遇到了一个小问题，促使我对隔离性有了新的认识和理解，本文就来讲讲事务的隔离性。

<!-- more -->

## 遇到的问题

这里就不对事务隔离性的定义再详细展开了，可以看我参考资料中的几篇博客，讲的都很到位。

现在开发中有这么一套业务逻辑：

```java
// 1. 查询数据库中是否有某条记录
// 2. 如果不存在，则通过 HTTP 调用另一个服务，执行一系列操作后，并插入该条数据
// 3. 再查一次，存在则继续接下来的业务
@Transactional
public void foo(){
    // # 1st query
    EntityA a = aService.findOneBy();
    if(a == null){
        // call other service and insert data
        httpUtils.doPost("http://ip:port/createA");
        // # 2nd query
        a = aService.findOneBy();
    }
}
```

**注：** 这边在设计上可能有一些问题，远程调用服务方法 `createA()`，其实直接将插入的数据 JSON 返回即可，不需要重复查询数据库，但是 `createA()` 这个方法默认是无返回的，由于某些原因无法做出改动。

乍一看这代码似乎并没有什么问题，逻辑也很简单。但实际上两次查询的结果是一样的，我们无法在成功执行 `createA()` 后得到新的数据。这是为什么呢？具体原因就是受到数据库隔离级别的影响。

## 事务的隔离性和 @Transactional

隔离性是关系型数据库事务 ACID 特征中的一种，意思是如果有多个事务并发执行，每个事务作出的修改必须与其他事务隔离。也就意味着事务间的操作相互不可见。同时为了权衡数据库性能和可靠性，SQL 标准中给出了集中隔离级别（不同的数据库实现方式不同）READ UNCOMMITED、READ COMMITED、REPEATABLE READ 和 SERIALIZABLE。

> - `RAED UNCOMMITED`：使用查询语句不会加锁，可能会读到未提交的行（Dirty Read）；
> - `READ COMMITED`：只对记录加记录锁，而不会在记录之间加间隙锁，所以允许新的记录插入到被锁定记录的附近，所以再多次使用查询语句时，可能得到不同的结果（Non-Repeatable Read）；
> - `REPEATABLE READ`：多次读取同一范围的数据会返回第一次查询的快照，不会返回不同的数据行，但是可能发生幻读（Phantom Read）；
> - `SERIALIZABLE`：InnoDB 隐式地将全部的查询语句加上共享锁，解决了幻读的问题；
>
> **MySQL 中默认的事务隔离级别就是 `REPEATABLE READ`，但是它通过 Next-Key 锁也能够在某种程度上解决幻读的问题。**

可通过以下 SQL 查看 MySQL 的隔离级别：

```sql
use performance_schema;
select * from global_variables where variable_name = 'tx_isolation';
```

再来看看 Spring 中的申明式事务注解的用法。`@Transactional` 中可以配置以下参数：

| 属性名             | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| name               | 当在配置文件中有多个 TransactionManager , 可以用该属性指定选择哪个事务管理器。 |
| propagation        | 事务的传播行为，默认值为 REQUIRED。                          |
| isolation          | 事务的隔离度，默认值采用 DEFAULT。                           |
| timeout            | 事务的超时时间，默认值为 \-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。 |
| read\-only         | 指定事务是否为只读事务，默认值为 false；为了忽略那些不需要事务的方法，比如读取数据，可以设置 read\-only 为 true。 |
| rollback\-for      | 用于指定能够触发事务回滚的异常类型，如果有多个异常类型需要指定，各类型之间可以通过逗号分隔。 |
| no\-rollback\- for | 抛出 no\-rollback\-for 指定的异常类型，不回滚事务。          |

`isolation = DEFAULT` 意味着 Spring 默认会采用数据中配置的隔离级别，也就是 `REPEATABLE READ`，并且 MySQL 也解决了幻读。因此两次的 query 操作查询到的记录都为 null。对于事务而言这显然是一个正常的结果，但是对于我们的业务逻辑而言就存在问题了。按照上面的代码，我们就是想 “幻读” 到其他事务作出的修改。

## 让事务可”幻读“

当我们清楚地知道某些数据出现不可重复读和幻读的现象时，其实也就没什么关系了。为了是的上面的代码能够正常运行，我们可以作出以上改动。

1. **直接修改 `foo()` 上的事务注解配置**

```
@Transactional(isolation = Isolation.READ_COMMITTED)
public void foo(){}
```

这样相当于在 MySQL 数据库的 `REPEATABLE READ` 隔离级别上降了一级，在 `READ_COMMITTED` 这个级别上会出现不可重复读和幻读的问题，也就意味着我们可以读到其他事务对数据库作出的修改，问题解决。

2. **将 `aService.findOneBy()` 方法以非事务方式运行或单独起一个事务**

```
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public findOneBy(){}
```

以非事务方式运行就可以读到其他事务的更新数据的。

```
@Transactional(propagation = Propagation.REQUIRES_NEW)
public findOneBy(){}
```

`Propagation.REQUIRES_NEW` 意思是创建一个新的事务，如果当前存在事务，则把当前事务挂起。这种方式相当于将 2 次查询操作不加入 `foo()` 的事务中，单独在自己的事务中运行，手动将事务串行化了，也就是 `1st query --> createA --> 2nd query` 依次进行，自然不存在并发事务，也就解决问题了。

## 总结

在大部分场景中，直接在 Service 层的方法上添加一个 `@Transactional` 就能让方法以事务的形式运行，能够很好的保证在出现异常的情况下有效的进行 rollback，防止产生脏数据。但是在某些特殊的场景下，还是需要手动细粒度的去控制事务的隔离级别和传播行为的，就比如文中描述到的这个场景。

隔离级别越高数据库的一致性越强，但性能越差。这点就需要在开发中去协调了，事务常见的问题脏读、不可重复读和幻读，除了脏读比较致命外，其余两个个人觉得只要是可控的，就都不是问题了。

还有一个问题，不知道能不能得到解答：SQL 标准中定义了事务具有隔离性，事务内部无法读到其他事务的操作结果，但如果 t1 读取了某条数据，并进行计算，此时 t2 修改了并且 commit，那么 t1 的计算结果不是全都会出现问题吗？不可重复读的意义在于哪？如果可以读到外部事务的更新，会出现什么问题？

## 参考资料
- [『浅入浅出』MySQL 和 InnoDB](https://draveness.me/mysql-innodb/)
- [『浅入深出』MySQL 中事务的实现](https://draveness.me/mysql-transaction/)
- [wikipedia - 事务隔离](https://zh.wikipedia.org/wiki/%E4%BA%8B%E5%8B%99%E9%9A%94%E9%9B%A2)
- [Innodb 中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)
- [透彻的掌握 Spring 中 @transactional 的使用](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html)
- [spring 的 @Transactional 注解详细用法](https://www.cnblogs.com/yepei/p/4716112.html)