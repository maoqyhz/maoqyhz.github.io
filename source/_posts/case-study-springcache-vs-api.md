---
title: 一个缓存使用的思考：Spring Cache VS Caffeine 原生 API
tags:
  - Spring Cache
  - Caffeine
categories:
  - 技术
copyright: true
date: 2019-12-09 21:54:38
---
最近在学习本地缓存发现，在 Spring 技术栈的开发中，既可以使用 Spring Cache 的注解形式操作缓存，也可用各种缓存方案的原生 API。那么是否 Spring 官方提供的就是最合适的方案呢？那么本文将通过一个案例来为你揭晓。

<!-- more -->

## Spring Cache

>  Since version 3.1, the Spring Framework provides support for transparently adding caching to an existing Spring application. The caching abstraction allows consistent use of various caching solutions with minimal impact on the code.

Spring Cache 和 slf4j、jdbc 类似，是由 Spring Framwork 提供的一个缓存抽象层，可以接入各种缓存解决方案来进行使用，通过 Spring Cache 的集成，我们只需要通过一组注解来操作缓存就可以了。目前支持的有 [Generic](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-generic)、[JCache (JSR-107)](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-jcache) 、[EhCache 2.x](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-ehcache2)、[Hazelcast](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-hazelcast)、[Infinispan](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-infinispan)、[Couchbase](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-couchbase)、[Redis](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-redis)、[Caffeine](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-caffeine)、[Simple](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider-simple)，几乎包含了主流的本地缓存方案。

其主要的原理就是向 Spring Context 中注入 Cache 和 CacheManager 这两个 bean，再通过 Spring Boot 的自动装配技术，会根据项目中的配置文件自动注入合适的 Cache 和 CacheManager 实现。

## 本地缓存方案

Java 技术栈中成熟的本地缓存方案已经有很多了，有大而全的 ehcache，也有后起之秀 Google Guava Cache。下面是常用的三大本地缓存方案的对比，引用自博客 [如何优雅的设计和使用缓存？](https://juejin.im/post/5b849878e51d4538c77a974a)

| 项目         | Ehcache                       | Guava Cache                       | Caffeine              |
| ------------ | ----------------------------- | --------------------------------- | --------------------- |
| 读写性能     | 好                            | 好，需要做淘汰操作                | 很好                  |
| 淘汰算法     | 支持多种淘汰算法, LRU,LFU,FIFO | LRU，一般                         | W\-TinyLFU, 很好      |
| 功能丰富程度 | 功能很丰富                    | 功能很丰富，支持刷新和虚引用等    | 功能和 Guava Cache 类似 |
| 工具大小     | 很大，最新版本 1\.4MB          | 是 Guava 工具类中的一个小部分，较小 | 一般，最新版本 644KB   |
| 是否持久化   | 是                            | 否                                | 否                    |
| 是否支持集群 | 是                            | 否                                | 否                    |

目前比较推荐的是 Caffeine，淘汰算法比较先进，并且得到 Spring Cache 的支持（新版的 Spring Cache 不再支持 Guava Cache）。下文的代码也是使用 Caffeine 的原生 API 的。

## 案例

使用过 Spring Cache 的人应该会发现，通过几个注解就能够轻松实现缓存的 CRUD 操作，并且替换其他的缓存方案不需要对代码进行改动吗，同时也不需要写例如下文的样板代码：

```java
{
    // 缓存命中
    if(cache.getIfPresent(key) != null){
        // todo
    }else{
        // 缓存未命中，IO 获取数据，结果存入缓存
        Object value = repo.getFromDB(key);
        cache.put(key,value);
    }
}
```

那学到这里，我就产生了疑惑，既然 Spring 出了缓存的注解化开发，并且大量的博客也都在往 Spring Cache 上引，那还是否需要用原生 API 呢？毕竟在 Spring Data JPA 出现后，我们的确很少关注后端 ORM 框架，也不再直接使用 Hibernate 了。

当我实现了项目中的一个需求，这个问题好像就豁然开朗了。

其实需求很简单，原本在本地 HashMap 中维护的一个映射表，由于后期需要频繁改动而放到了数据库中。但由于数据量并不大且不配置映射表时，数据保持不变，因此既然在学习缓存，就想把它加进去。那么现在需要做的就是：

1. 一个读取映射表全表的方法 `aliasMap()`。并缓存数据到 Caffeine。
2.  一个支持映射记录 CRUD 操作的页面，且修改映射表时，更新缓存。

```java
@Cacheable(value = "default", key = "#root.methodName")
@Override
public Map<String, String> aliasMap() {
	return getMapFromDB();
}
```

由于 Spring Cache 的注解一般是添加在类或者方法上的，换而言之，缓存的是方法返回的对象。显然，通过某个方法来触发另一个缓存中的对象的更新是行不通的。这样是否意味着 Spring Cache 无法实现了呢？仔细去看一下 Spring Cache 的原理，其实还是可行的。

Spring Cache 会向 Spring Context 中注入 Cache 和 CacheManager 这两个 bean，再通过 Spring Boot 的自动装配技术，根据项目中的配置文件自动注入合适的 Cache 和 CacheManager 实现。再看到 CaffeineCacheManager 的源码：

```java
public class CaffeineCacheManager implements CacheManager {
    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap(16);
    private boolean dynamic = true;
    private Caffeine<Object, Object> cacheBuilder = Caffeine.newBuilder();
    @Nullable
    private CacheLoader<Object, Object> cacheLoader;
    private boolean allowNullValues = true;
}
```

显然，缓存是存在 cacheMap 这样一个 ConcurrentHashMap 中，那只要我们能够手动去获取到这个 bean 的实例去操作它，那么这个需求就可以实现了，代码如下：

```java
@Autowired
private CacheManager cacheManager;
@Cacheable(value = "default", key = "#root.methodName")
@Override
public Map<String, String> aliasMap() {
    return getMapFromDB();
}

private Map<String, String> getMapFromDB() {
    Map<String, String> map = new HashMap<>();
    List<PartAlias> list = repository.findAll();
    list.forEach(x -> map.put(x.getAlias(), x.getName()));
    return map;
}

@Override
public PartAlias saveOrUpdateWithCache(PartAlias obj) {
    PartAlias partAlias = repository.saveAndFlush(obj);
    Cache cache = cacheManager.getCache("default");
    cache.clear();
    cache.put("aliasMap", getMapFromDB());
    return partAlias;
}
```

经过测试，上面的代码是可行的。显然，遇到一些稍微复杂的需求，仅仅依靠 Spring Cache 的注解是远远不够的，我们需要自己去操作 cache 对象。如果使用原生 API 就非常简单了，能应对不同的需求。

### What's More

上面的需求，Spring Cache 尚且还是能够处理的，但是如果要实现数据的自动加载和刷新呢？现在 Spring Cache 并不能够很好的支持。

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1024
    cache-names: cache1,cache2
```

上面的代码是用来配置 cache 的，结合上文 CaffeineCacheManager 的源码，我们可以知道，Spring Cache 的配置是全局的，也就是说例如最大条数、过期时间等参数是为全体缓存进行设置的，无法单独为某个缓存设置。而在 Caffeine 中用于数据加载和刷新的 `CacheLoader` 也是 `CaffeineCacheManager` 这个 bean 共有的，因此也就失去存在的意义，毕竟每个缓存的加载和数据刷新的方式是不可能相同的。

因此，在遇到复杂场景下， 还是得上原生 API 的，Spring Cache 就显得心有余而力不足了。笔者也写个一个工具类，可以全局使用缓存。

```java
@Component
public class CaffeineCacheManager {
    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);

    /**
     * 缓存创建
     *
     * @param cacheName
     * @param cache
     */
    public void createCache(String cacheName, Cache cache) {
        cacheMap.put(cacheName, cache);
    }

    /**
     * 缓存获取
     *
     * @param name
     * @return
     */
    public synchronized Cache getCache(String name) {
        Cache cache = this.cacheMap.get(name);
        if (cache == null) {
            throw new IllegalArgumentException("No this cache.");
        }
        return cache;
    }

    @Autowired
    private static CaffeineCacheManager manager;
    public static void main(String[] args) {
        manager.createCache("default", Caffeine.newBuilder()
                .maximumSize(1024)
                .build());
        Cache<String, Object> cache = manager.getCache("default");
        // TODO
    }
}
```

当然，再来提一提，既然是 Spring 的套路，总是会给开发者留一条后路的，如果愿意折腾的，可以阅读 CacheManager 的代码，再根据自己需求重新实现，从而管理自己的 cache 实例。

## 总结

本文不是一篇介绍 Spring Cache 和 Caffeine 用法的文章（有需要可以阅读参考文献），而是在探讨 Spring Cache 和 Caffeine 的原生 API 的使用场景。显然，Spring 全家桶有时未必是最优的解决方案（有能力重写的另当别论了）！所以也希望网上有更多的博客可以 focus on 框架本身的使用，而不是千篇一律的各种集成到 Spring xxx。

## 附录

### yaml 配置

```YAML
initialCapacity: # 初始的缓存空间大小
maximumSize: # 缓存的最大条数
maximumWeight: # 缓存的最大权重
expireAfterAccess: # 最后一次写入或访问后经过固定时间过期
expireAfterWrite: # 最后一次写入后经过固定时间过期
refreshAfterWrite: # 创建缓存或者最近一次更新缓存后经过固定的时间间隔，刷新缓存
weakKeys: # 打开 key 的弱引用
weakValues:  # 打开 value 的弱引用
softValues: # 打开 value 的软引用
recordStats: # 开发统计功能
```

### 原理篇

#### 合理使用缓存

缓存的目的主要是为了降低主主数据库的压力，服务可以直接从缓存中获取数据，从而提高响应速度，让原本有限的资源可以服务更多的用户。

从工程的角度来说，缓存的引入并不是盲目的，如果主数据库压力不大的情况，并不需要添加缓存。多添加一个数据中间件显然也会增加维护的成本，而且在实际使用过程中还会存在一些，例如缓存击穿、缓存雪崩等问题。

#### 基本概念

- 命中率。 返回正确结果数 / 请求缓存次数， 命中率越高，表明缓存的使用率越高。
- 最大元素。缓存中可以存放元素的最大数目， 一旦超过，会通过合适的策略进行清空操作。
- 清空策略：FIFO、LFU、LRU

#### 缓存类型

缓存根据存储的方式可以分成本地缓存和分布式缓存。

- 本地缓存：本地缓存一般指的是缓存在应用进程内部的缓存。以 Java 技术栈为例，可是自己实现一个 HashMap 作为数据缓存，也可以直接使用现成的缓存方案，例如 ehcache、caffeine 等。
- 分布式缓存：缓存和应用环境分离，会单独存放在自己的服务器或集群里，且多个应用可直接的共享缓存。 常见的缓存方案有 MemCache 和 Redis 等。

这一节主要是让大家对缓存有一个基本的认识，缓存不是一种具体的技术，而是一种通用的技术方案，如何选择合适的缓存方案集成到自己的项目中去并且如何解决引入缓存后产生的一些经典问题，不是本文讨论的重点。有关于缓存的详细介绍和选型可以参考：

- [美团技术团队——缓存那些事]( https://cloud.tencent.com/developer/article/1058203 )
- [如何优雅的设计和使用缓存？](https://juejin.im/post/5b849878e51d4538c77a974a)

## 参考文献

- [spring-framework-cache](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/integration.html#cache)
- [spring-boot-cache](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching)
- [caffeine/wiki](https://github.com/ben-manes/caffeine/wiki)
- [如何优雅的设计和使用缓存？](https://juejin.im/post/5b849878e51d4538c77a974a)
- [你应该知道的缓存进化史](https://juejin.im/post/5b7593496fb9a009b62904fa)
- [Caffeine 缓存](https://www.jianshu.com/p/9a80c662dac4)
- [Spring Boot 缓存实战 Caffeine](https://www.jianshu.com/p/c72fb0c787fc)
- [Spring Boot 2.X(七)：Spring Cache 使用](https://juejin.im/post/5da52d55f265da5b5b6c6d3)