---

title: Bean装配，从Spring到Spring Boot
date: 2019/8/20 24:00:00
categories:
  - 技术
tags:
  - Spring Boot
  - Spring
  - IoC
copyright: true
---

本文旨在厘清从Spring 到Spring Boot过程中，Bean装配的过程。

<!--more-->

自从用上Spring Boot，真的是一直用一直爽，已经完全无法直视之前Spring的代码了。**约定大于配置**的设计理念，使得其不需要太多的配置就能开箱即用。但是由于其便捷性，也就意味着掩盖了许多细节的部分，使得直接学习Spring Boot的开发者只会用，而不清楚内部的实现流程。最近刚好有空，重新回顾了一下Spring的相关内容，并且梳理了有关于Bean装配的一些用法，描述从过去的Spring开发，到现在的Spring开发 Boot在Bean装配上的变化和进步。

## 从SSM的集成谈到Bean的装配

在学习初期，我想每个人都会去看一些博客，例如“Spring Spring MVC Mybatis整合”。一步一步整合出一个ssm项目。那个时候的我是没有什么概念的，完全就是，跟着步骤走，新建配置文件，把内容贴进来，然后run，一个基本的脚手架项目就出来了。回顾一下，基本是以下几步：

1. 建立maven web项目，并在pom.xml中添加依赖。
2. 配置web.xml，引入spring-*.xml配置文件。
3. 配置若干个spring-*.xml文件，例如自动扫包、静态资源映射、默认视图解析器、数据库连接池等等。
4. 写业务逻辑代码（dao、services、controller）

后期可能需要用到文件上传了，再去xml中配置相关的节点。在开发中，基本也是遵循一个模式——三层架构，面向接口编程。类如果是Controller的就加一个@Controller；是Services的就加一个@Services注解，然后就可以愉快的写业务逻辑了。

Spring是什么？那个时候的理解，像这样子配置一下，再加上几个注解，就是Spring。或者说，Spring就是这么用的。

随着学习的深入，对Spring就有更深的理解了。**在Spring当中，一切皆为Bean。**可以说Bean是组成一个Spring应用的基本单位。Spring（这里是狭义的概念，指Spring Core）中最核心部分就是对Bean的管理。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
                         http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context-3.2.xsd
                        http://www.springframework.org/schema/mvc
                        http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <context:annotation-config/>

    <context:component-scan base-package="com.zjut.ssm.controller">
        <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <bean id="defaultViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <property name="maxUploadSize" value="20971500"/>
        <property name="defaultEncoding" value="UTF-8"/>
        <property name="resolveLazily" value="true"/>
    </bean>
</beans>
```

让我们再次看一下Spring MVC的配置文件，除了一些参数外，还有两个bean节点，注入InternalResourceViewResolver来处理视图，注入CommonsMultipartResolver来处理文件上传。这个时候，如果需要集成Mybatis一起工作，类似的，注入相关的Bean就可以了。Mybatis最核心的Bean就是SqlSessionFactory，通过创建Session来进行数据库的操作。在不使用Spring时，可以通过加载XML，读入数据库信息，进行创建。

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

显然，上面的代码是不符合Spring的思想的。为了达到松耦合，高内聚，尽可能不直接去new一个实例，而是通过DI的方式，来注入bean，由Spring IoC容器进行管理。Mybatis官方给到一个MyBatis-Spring包，只需添加下面的Bean就可以组织Mybatis进行工作了（创建sqlSessionFactory，打开Session等工作），关于Mybatis的内容，这里不展开了。

```xml
<beans>
    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="${jdbc.driver}"/>
        <property name="jdbcUrl" value="${jdbc.url}"/>
        <property name="user" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="typeAliasesPackage" value=""/>
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    </bean>
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <property name="basePackage" value=""/>
    </bean>
</beans>
```

那么，现在我们就知道了，**通过XML配置Spring，集成各种框架的实质，就是Bean装配的。**每一个框架都是由N多个Bean构成的，如果需要使用它，就必须根据框架的要求装配相应的Bean。装配成功的Bean由Spring IoC容器统一管理，就能够正常进行工作。

## Bean的装配

具体Bean的装配方式，发展到现在也已经有很多种了，从过去的XML到Java Config，再到现在Spring Boot的Auto Configuration，是一种不断简化，不断清晰的过程。

Bean装配一般分为三步：注册、扫描、注入。

### 由XML到Java Config

XML配置Bean早已成为明日黄花了，目前**更常见的是使用Java Config和注解来进行Bean的装配**。当然，偶尔也能看到它的身影，例如用于ssm框架的集成的spring-*.xml。

Java Config的优势如下：

1. Java是类型安全的，如果装配过程中如果出现问题，编译器会直接反馈。
2. XML配置文件会随着配置的增加而越来越大，不易维护管理。
3. Java Config更易于重构和搜索Bean。

因此下面主要讲都是基于Java的配置方法。基本流程如下：

```java
// 注册
@Configuration
public class BeanConfiguration {
    @Bean
    public AtomicInteger count() {
        return new AtomicInteger();
    }
}
//或者 
@Componment
public class Foo{}

// 扫描
@ComponentScan(basePackages={})
@Configuration
public class BeanConfiguration {}

// 注入
@Autowired
private AtomicInteger c;
```

下面详细展开。

Java Config注册Bean，主要分为两类，注册非源码的Bean和注册源码的Bean。

#### 注册非源码的Bean

非源码的Bean，指的是我们无法去编辑的代码，主要是引入外部框架或依赖，或者使用Spring的一些Bean。这些Bean的配置一般采用Java文件的形式进行声明。

新建一个使用`@Configuration`修饰的配置类，然后使用`@Bean`修饰需要创建Bean的方法，具体的可指定value和name值（两者等同）。示例如下：

```java
@Configuration
public class BeanConfiguration {
    @Scope("prototype")
    @Bean(value = "uploadThreadPool")
    public ExecutorService downloadThreadPool() {
        return Executors.newFixedThreadPool(10);
    }
}
```

其中需要注意的是：

1. Bean默认是以单例的形式创建的，整个应用中，只创建一个实例。如果需要，每次注入都创建一个新的实例，可添加注解`@Scope("prototype")`。对于这个例子而言，默认创建的线程池是单例的，在应用的任何一个地方注入后使用，用到的都是同一个线程池（全局共享）；加上`@Scope("prototype")`后，在每一个Controller中分别注入，意味着，每一个Controller都拥有各自的线程池，各自的请求会分别提交到各自的线程池中。
2. 注入多个相同类型的Bean时，手动指定name或value值加以区分。或通过`@Primary`，标出首选的Bean。

#### 注册源码的Bean

源码的Bean，指的是我们自己写的代码，一般不会以@Bean的形式装配，而是使用另外一系列具有语义的注解。（@Component、@Controller、@Service、@Repository）添加这些注解后，该类就成为Spring管理的组件类了，列出的后三个注解用的几率最高，基本已经成为样板注解了，Controller类添加@Controller，Service层实现添加@Service。

下面展示一个例子，通过Spring声明自己封装的类。

```java
@Scope("prototype")
@Component(value = "uploadThread")
public class UploadTask implements Runnable {
    private List<ByteArrayInputStream> files;
    private List<String> fileNameList;
    private PropertiesConfig prop = SpringUtil.getBean(PropertiesConfig.class);

    // 如果直接传入MutiPartFile，文件会无法存入，因为对象传递后spring会将tmp文件缓存清楚
    public UploadThread(List<ByteArrayInputStream> files, List<String> fileNameList)     {
        this.files = files;
        this.fileNameList = fileNameList;
    }

    @Override
    public void run() {
        for (int i = 0; i < files.size(); ++i) {
            String fileName = fileNameList.get(i);
            String filePath = FileUtils.generatePath(prop.getImageSavePath(),fileName);
            FileUtils.save(new File(filePath), files.get(i));
        }
    }
}
```

接着上面的线程池讲，这里我们实现了一个task，用来处理异步上传任务。在传统JUC中，我们一般会这么写代码：

```
private ExecutorService uploadThreadPool = Executors.newFixedThreadPool(10);
uploadThreadPool.submit(new UploadTask(fileCopyList, fileNameList));
```

在Spring中，我觉得就应该把代码写的更Spring化一些，因此添加@Component使之成为Spring Bean，并且线程非单例，添加@Scope注解。重构后的代码如下：

```java
@Resource(name = "uploadThreadPool")
private ExecutorService uploadThreadPool;

@PostMapping("/upload")
public RestResult upload(HttpServletRequest request) {
    uploadThreadPool.submit((Runnable) SpringUtils.getBean("uploadThread", fileCopyList, fileNameList));
}
```

Bean的注入在下节会仔细讲。其实这样写还有零一个原因，**非Spring管理的Bean一般是无法直接注入Spring Bean的**。如果我们需要在UploadTask中实现一些业务逻辑，可能需要注入一些Services，最好的做法就是讲UploadTask本身也注册成Spring Bean，那么在类中就能够使用@Autowired进行自动注入了。

额外需要注意的是：**由于线程安全的一些原因，线程类是无法直接使用@Autowired注入Bean的。**一般会采用`SpringUtils.getBean()`手动注入。


### 自动扫描

在配置类上添加` @ComponentScan`注解。该注解默认会扫描该类所在的包下所有的配置类，特殊的包可以配置`basePackages`属性。Spring扫描到所有Bean，待注入就可以使用了。

### Bean的注入

对于一些不直接使用的Bean，注册到Spring IoC容器后，我们是不需要手动去注入的。例如前面提到Mybatis的三个Bean。我们只需要根据文档进行使用，创建Mapper接口，并使用@Mapper修饰，在调用具体的查询方法时，Mybatis内部会进行Bean的注入，Open一个Session进行数据库的操作。

对于我们需要使用到的Bean，就需要注入到变量中进行使用。常用Bean注入的方式有两种。

1. **注解注入（@Autowired和@Resource）。**

`@Autowired`是Bean注入最常用的注解，默认是通过**byType**的方式注入的。也就是说如果包含多个相同类型的Bean，是无法直接通过@Autowired注入的。这个时候需要通过@Qualifier限定注入的Bean。或者使用@Resource。`@Resource`是通过**byName**的方式注入，直接在注解上标明Bean的name即可。

```java
public class Foo{
    // 正常字段注入
    @Autowired
    private AtomicInteger c;
  
    // 正常构造器注入
    private final AtomicInteger c;
    @Autowired
    public Foo(AtomicInteger c){this.c = c;}
  
    // 歧义加上@Qualifier
    @Autowired
    @Qualifier("count")
    private AtomicInteger c;  
  
    // 歧义直接使用@Resource（与前一种等同）
    @Resource("count")
    private AtomicInteger c;  
}
```

2. **调用Application Context的getBean方法。**

推荐使用注解注入，但是在默写特殊情况下，需要使用getBean()方法来注入Bean。例如之前讲到的多线程环境。具体实现可见附录，实现ApplicationContextAware，可以封装成一个工具类。

### SSM集成的Java版

由于Java Config的优势，框架集成工作很多选择不用XML配置了。例如之前在xml中配置Spring MVC的ViewResolver，在Java中就可以这样去注入：

```java
@Configuration
@EnableWebMvc
public class WebConfig extends WebMvcConfigurerAdapter {
    @Bean
    public ViewResolver viewResolver(){
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/view/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
}
```

有兴趣的，可以自己去使用Java Config集成一下，其实需要配置的东西都是一致的，只是在Java中，有的是要注册对应的Bean，有的需要实现对应的接口罢了。由XML到Java，仅仅只是配置语言的改变，真正解放程序员，提高生产力是Spring Boot。基于约定大于配置的思路，通过Auto Configuration，大规模减少了一些缺省Bean的配置工作。

## Spring Boot Magic

### Auto Configuration

接下来我们看一下，基于Spring Boot的SSM集成需要配置的内容。

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: fur@6289
mybatis:
  type-aliases-package: com.fur.mybatis_web.mapper
  mapper-locations: classpath:mapping/*.xml
```

用过Spring Boot的都知道，上面的是Spring Boot的配置文件`application.yml`。只需要在pom.xml添加对应依赖，在yml里配置必要的信息。（框架再智能也不可能知道我们的数据库信息吧:smile:）之前看到的CommonsMultipartResolver、SqlSessionFactoryBean等再也不需要手动去装配了。

Spring Boot和传统SSM的不同之处在于：

1. 添加的maven依赖不同，由`mybatis`和`mybatis-spring`变成了`mybatis-spring-boot-starter`
2. 多了@SpringBootApplication，少了@ComponentScan

这里也不对Spring Boot Starter进行介绍了，只要知道这就是一个新的dependency，包含自动配置和其他需要的依赖就行了。在过去Spring中引入的依赖中，如果Boot实现了对应的starter，应该优先使用Starter。

下面我们就跟一下源码，了解一下Spring Boot到底是如何实现Auto Configuration的。Take it easy，我们不会逐行去分析源码，只是梳理下大概的流程。

顺着思路，首先看到的是@SpringBootApplication注解中的内容，显然这是一个复合注解，主要包含@SpringBootConfiguration、@EnableAutoConfiguration和@ComponentScan。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}
```

- @SpringBootConfiguration，内部是一个@Configuration，跟之前一样，注解修饰的类是一个Spring配置类。
- @ComponentScan，也不陌生。集成在@SpringBootApplication注解中，创建项目时默认添加，我们无需手动开启Bean扫描了。
- @EnableAutoConfiguration，这个就是Auto Configuration的关键。简而言之，加上这个注解，Spring会自动扫描引入的依赖（classpath下的jar包）中需要Auto Configuration的项目，并且根据yml进行自动配置工作。

再看@EnableAutoConfiguration，发现内部import一个`AutoConfigurationImportSelector`类，这个类基本上就是处理Auto Configuration工作的核心类了。

```java
// AutoConfigurationImportSelector.java
/**
 * Return the auto-configuration class names that should be considered. By default
 * this method will load candidates using {@link SpringFactoriesLoader} with
 * {@link #getSpringFactoriesLoaderFactoryClass()}.
 * @param metadata the source metadata
 * @param attributes the {@link #getAttributes(AnnotationMetadata) annotation
 * attributes}
 * @return a list of candidate configurations
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
            + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

看到上面的这个方法，`getCandidateConfigurations()`的作用就是获取需要自动配置的依赖信息，核心功能由SpringFactoriesLoader的`loadFactoryNames()`方法来完成。

```java
// SpringFactoriesLoader.java
public final class SpringFactoriesLoader {
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    /**
     * Load the fully qualified class names of factory implementations of the
     * given type from {@value #FACTORIES_RESOURCE_LOCATION}, using the given
     * class loader.
     * @param factoryClass the interface or abstract class representing the factory
     * @param classLoader the ClassLoader to use for loading resources; can be
     * {@code null} to use the default
     * @throws IllegalArgumentException if an error occurs while loading factory names
     * @see #loadFactories
     */
    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
    }
}
```

这里简单说一下流程，有兴趣的可以看下这个类的源码。SpringFactoriesLoader会扫描classpath下的所有jar包，并加载META-INF/spring.factories下的内容。我们以mybatis-spring-boot-starter为例，里面有个依赖mybatis-spring-boot-autoconfigure，可以看到存在META-INF/spring.factories。

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/bean-definition-from-spring-to-springboot/1566182027899.png)

内容如下：

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

接下来，会获取到key为`org.springframework.boot.autoconfigure.EnableAutoConfiguration`的值。也就是Mybatis具体的自动配置类。可以看到在包`org.mybatis.spring.boot.autoconfigure`下，有`MybatisAutoConfiguration`和`MybatisProperties`，前者是自动配置类，后者是用于配置的一些参数，对应在yml中的mybatis节点下的键值对。下面贴一些源码看看：

```java
// MybatisAutoConfiguration.java
@Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration implements InitializingBean {
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception     {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        // 省略
        return factory.getObject();
    }
}
```

这里就看到了熟悉的SqlSessionFactory，之前我们需要手动注入，现在通过`@ConditionalOnMissingBean`，当Spring容器中不存在这个Bean时，就会为我们自动装配。

下面的流程图就是简单描述了整个自动配置的流程。

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/bean-definition-from-spring-to-springboot/auto%20config.png)

### Syntactic sugar

其实就是复合注解啦，除了约定大于配置的自动装配外，Spring Boot通过一个大的复合注解@SpringBootApplication，把@ComponentScan和@SpringBootConfiguration也包在里面，使得启动类直接就可以作为配置类使用，并且减少了Bean扫描这一步。

## 总结

本文不算是一篇完整的Spring IoC教程，只是梳理了一下从Spring 到Spring Boot一路走来，在Bean装配上的用法和改变。可以看到这是一个不断简化，不断进步的过程，但是核心依然不变，因此即使当下转到Spring Boot 进行开发，在遇到一些复杂的业务上，任然需要用到Spring IoC相关的技术点，例如控制Bean的作用域，条件化Bean等等。

## 参考文献

- [第5章 Spring Boot自动配置原理](https://www.jianshu.com/p/346cac67bfcc)
- [Spring Boot 为什么这么火？](http://www.ityouknow.com/springboot/2019/06/03/spring-boot-hot.html)
- [《Spring 实战》](https://www.manning.com/books/spring-in-action-fourth-edition)

## 附录

### SpringUtils.java

```java
@Component
public class SpringUtils implements ApplicationContextAware {
    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        if (SpringUtil.applicationContext == null) {
            SpringUtil.applicationContext = applicationContext;
        }
    }

    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    public static Object getBean(String name) {
        return getApplicationContext().getBean(name);
    }

    public static <T> T getBean(Class<T> clazz) {
        return getApplicationContext().getBean(clazz);
    }

    public static <T> T getBean(String name, Class<T> clazz) {
        return getApplicationContext().getBean(name, clazz);
    }

    public static <T> T getBean(String name, Object... args) {
        return (T) getApplicationContext().getBean(name, args);
    }
}
```

### 非Spring Bean中获取Spring Bean的方法

在非Spring Bean中获取Spring Bean，需要改造SpringUtils.java，去掉@Component，并且不需要实现接口了，手动注入Spring Context。

```java
public class SpringUtils {
    private static ApplicationContext applicationContext;
    public static void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        if (SpringUtil.applicationContext == null) {
            SpringUtil.applicationContext = applicationContext;
        }
    }
    // 余下代码如上
}
```
```java
public class SkeletonApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(SkeletonApplication.class, args);
        SpringUtils.setApplicationContext(context);
    }
}
```
