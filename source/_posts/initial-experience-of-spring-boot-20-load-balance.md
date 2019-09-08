---
title: Spring Cloud 负载均衡初体验
tags:
  - Spring Boot
  - Java
  - Spring Cloud
  - Load Balance
  - Eureka
  - Ribbon
  - Feign
  - Hystrix
  - Microservices
categories:
  - 技术
copyright: true
date: 2019-08-28 22:43:16
---

使用 Spring Cloud Netflix 组件 Eureka 和 Ribbon 构建单注册中心的负载均衡服务。
<!-- more -->

Spring Cloud 是基于 Spring 的微服务技术栈，可以这么概括吧，里面包含了很多例如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等组件，可以通过 Spring Boot 的形式进行集成和使用。

目前，项目中有这么个需求，Spring Boot 做一个 web 服务，然后调用 TensorFlow 模型得到结果。但是由于 TensorFlow GPU 版，不支持多线程多引擎，所以只能采用多进程的方式去进行调度，所以需要做一个负载均衡。负载均衡的话，可以分为客户端和服务端负载均衡。我目前还没能领悟到有什么不同，毕竟整体的架构都是一样的，如下如图。其中客户端均衡负载的代表是 Spring Cloud 的 Ribbon，服务端负载均衡代表是 Nginx。

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/initial-experience-of-spring-boot%20-load-balance/load-balance.png)

由于项目的压力并不大，日平均请求约 5000 左右，因此就采用 Spring Cloud 中的组件进行客户端负载均衡。主要用到的就是 Spring Cloud 和 Eureka。很多博客中会也看到 Ribbon 的身影。其实他们都是 Spring Cloud Netflix 中的组件，用来构建微服务。本文所讲的例子，也可以看作是一个微服务，把原来一个的算法服务拆成了若干个小服务。之前讲到的 Spring Cloud 的各种组件也都是为了使得这些独立的服务能够更好的管理和协作。

回到负载均衡，一个使用 Spring Boot 搭建的客户端负载均衡服务，其实只需要 Rureka 这一个组件就够了。

Eureka 是 Spring Cloud Netflix 当中的一个重要的组件，用于服务的注册和发现。Eureka 采用了 C-S 的设计架构。具体如下图，Eureka Server 作为一个注册中心，担任服务中台的角色，余下的其他服务都是 Eureka 的 Client。所有的服务都需要注册到 Eureka Server 当中去，由它进行统一管理和发现。Eureka Server 作为管理中心，自然，除了注册和发现服务外，还有监控等其他辅助管理的功能。

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/initial-experience-of-spring-boot%20-load-balance/spring-load-balance.png?x-oss-process=style/resize_small_80)

具体从负载均衡的角度来讲：

1. Eureka Server——提供服务注册和发现
2. Service Provider——服务提供方，一般有多个服务参与调度
3. Service Consumer——服务消费方，从 Eureka 获取注册服务列表，从而能够消费服务，也就是请求的直接入口。

## 服务搭建

下面主要实战一下负载均衡服务搭建。

正如上面所说，一套完整的负载均衡服务，至少需要三个服务。

### 1.注册中心——Eureka Server

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/initial-experience-of-spring-boot%20-load-balance/d1.png)

直接通过 IDEA 创建一个包含 Eureka Server 的 Spring Boot 项目，直接引入所需的 dependency。主要是 `spring-cloud-dependencies` 和 `spring-cloud-starter-netflix-eureka-server`。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

在启动类上添加注解 `@EnableEurekaServer`。

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

  在 yml 中配置 eureka。

```yaml
server:
  port: 8080
eureka:.
    instance:
    prefer-ip-address: true
  client:
    # 表示是否将自己注册到 Eureka Server，默认为 true
    register-with-eureka: false
    # 表示是否从 Eureka Server 获取注册信息，默认为 true
    fetch-registry: false
    service-url:
      # 默认注册的域，其他服务都往这个 url 上注册
      defaultZone: http://localhost:${server.port}/eureka/
```

由于目前配置的是单节点的注册中心，因此 `register-with-eureka` 和 `fetch-registry` 都设为 false，不需要把自己注册到服务中心，不需要获取注册信息。

启动工程后，访问：http://localhost:8080/，就能看到一个图形化的管理界面，目前没有注册任何服务。

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/initial-experience-of-spring-boot%20-load-balance/eureka-dashboard.png)

### 2.服务提供方——Service Provider

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/initial-experience-of-spring-boot%20-load-balance/d2.png)

在 yml 中配置 Eureka。

```yaml
eureka:
  instance:
    prefer-ip-address: true    # 以 IP 的形式注册
    # 默认是 hostname 开头的，修改成 ip 开头
    instance-id: \${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
  client:
    serviceUrl:
      defaultZone: http://localhost:8080/eureka/    # 注册到之前的服务中心
server:
  port: 8084
spring:
  application:
    name: services-provider    # 应用名称
```

默认，Eureka 是通过 hostname 来注册到 Eureka Server 上的，由于后面可能涉及到多节点的配置，hostname 可能不如 ip 方便管理，所以将 prefer-ip-address 设为 true，通过 ip 注册，并修改 instance-id 格式为：\${spring.cloud.client.ip-address}:\${spring.application.name}:\${server.port}。注意 ip-address 为短线连接。

然后写一个简单的 web 服务做测试。

```java
@RestController
public class ServicesController {
    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }
}
```

  然后运行工程。

### 3.服务消费方——Service Consumer

新建项目同 2，然后配置 Eureka：

```yaml
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
  client:
    serviceUrl:
      defaultZone: http://localhost:8080/eureka/
server:
  port: 8085
spring:
  application:
    name: services-consumer
```

和之前一样，注意区分端口号和 name 就可以了。

再在启动类，配置 RestTemplate 用于负载均衡的服务分发。这是服务消费的一种基本方式，下面还会介绍第二种 Feign。

```java
@SpringBootApplication
public class ServicesConsumerApplication {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    public static void main(String[] args) {
        SpringApplication.run(ServicesConsumerApplication.class, args);
    }
}
```

在 Controller 中的调用

```java
@RestController
public class ConsumerController {
    // services-provider 为服务 provider 的 application.name，负载均衡要求多个服务的 name 相同
    private final String servicesUrl = "http://services-provider/";
    private final RestTemplate restTemplate;

    public ConsumerController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @RequestMapping("/")
    public String home() {
        return restTemplate.getForObject(servicesUrl,String.class);
    }
}
```

然后运行工程，可以发现两个服务都已经成功注册到注册中心。

![](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/initial-experience-of-spring-boot%20-load-balance/instances.png)

有的博客中可能提到需要添加，@EnableDiscoveryClient 或 @EnableEurekaClient，而实际上，官方文档已经明确，只要 spring-cloud-starter-netflix-eureka-client 在 classpath 中，应用会自动注册到 Eureka Server 中去。

> By having spring-cloud-starter-netflix-eureka-clienton the classpath, your application automatically registers with the Eureka Server. Configuration is required to locate the Eureka server.
> https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.0.M1/

直接调用 localhost:8085，就显示 hello world，说明成功！如果不相信的话，可以起两个 provider 服务，然后分别输出不同的字符串来验证负载均衡是否成功。

```
2019-08-13 15:08:18.689  INFO 14100 --- [nio-8085-exec-4] c.n.l.DynamicServerListLoadBalancer: DynamicServerListLoadBalancer for client services-provider initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=services-provider,current list of Servers=[192.168.140.1:8084]}
```

简化后的日志如上，可以看到，注解 @LoadBalanced 的负载均衡功能是通过 DynamicServerListLoadBalancer 来实现，这是一种默认的负载均衡策略。获取到服务列表后，通过 Round Robin 策略（服务按序轮询从 1 开始，直到 N）进行服务分发。

至此，本地的负载均衡服务就搭建完成了。

## 服务消费 Feign 与断路器 Hystrix

如果是初体验的话可以忽略这一节。添加这一节的目的是为了服务的完整性。在实际开发中，Feign 可能用到的更多，并且多会配合 Hystrix。

Feign 也是一种服务消费的方式，采用的是申明式的形式，使用起来更像是本地调用，它也是 Spring Cloud Netflix 中的一员。

Hystrix，简而言之就是一种断路器，负载均衡中如果有服务出现问题不可达后，通过配置 Hystrix，可以实现服务降级和断路的作用，这个功能 Eureka 默认是不提供的。因此，之前说了，为了负载均衡的完整性，需要添上它，否则，Ribbon 依然会将请求分发到有问题的服务上去。

那么要使用他们，首先需要添加依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

在启动类上加上 `@EnableFeignClients`。

调用的时候需要实现接口。

```java
@FeignClient(value = "services-provider")
public interface IRestRemote {
    @GetMapping("/")
    String home();
}
```

最后在 Controller 中注入接口，直接调用就可以了。

```java
@RestController
public class ConsumerController {
    private final IRestRemote remote;

    public ConsumerController(IRestRemote remote) {
        this.remote = remote;
    }

    @RequestMapping("/")
    public String home() {
        return remote.home();
    }
```

相对于 RestTemplate，Fegin 使用起来更简单，不需要去构造参数请求了，并且 Feign 底层集成了 Ribbon，也不需要显示添加 `@LoadBalanced` 注解了。同时 Feign 也可以直接使用下面要讲的 Hystrix。

Hystrix 的功能很多，远不止断路器一种，需要详细了解可看别的博客。这里就讲一下 Feign 如何集成 Hystrix 工作的。

下面先描述一下具体场景。假如现在有两个负载均衡服务，其中有一个挂了或出现异常了，如果没有断路器，进行服务分发的时候，仍然会分配到。如果开启 Hystrix，首先会进行服务降级（FallBack），也就是出现问题，执行默认的方法，并在满足一定的条件下，熔断该服务。

在原来 @FeignClient 的基础上添加 `fallback` 参数， 并实现降级服务。

```java
@FeignClient(value = "services-provider",fallback = RestRemoteHystrix.class)
public interface IRestRemote {
    @GetMapping("/")
    String home();
}
```

```JAVA
@Component
public class RestRemoteHystrix implements IRestRemote {
    @override
    public String home() {
        return "default";
    }
}
```

最后，在配置文件中开启 Feign 的 Hystrix 开关。

```yaml
feign:
  hystrix:
    enabled: true
```

下面就可以测试了，沿着之前的例子，分别开启两个服务，并输出不同的文本，当关闭一个服务后，再请求会发现，分配到关闭的那个服务时，会显示 “default”，请求多次后发现，该服务不可用，说明断路器配置成功！**欲知更多，可以阅读参考文献 5-6。**

## 特别注意

1.**多节点的配置**

如果不是单机，则需要修改部分字段，具体如下注释：

```yaml
eureka:
  instance:
    prefer-ip-address: true     # 以 IP 的形式注册
    # 主机的 ip 地址（同样考虑集群环境，一个节点可能会配置多个网段 ip，这里可指定具体的）
    ip-address:
    # 主机的 http 外网通信端口，该端口和 server.post 不同，比如外网需要访问某个集群节点，直接是无法访问 server.post 的，而是需要映射到另一端口。
    # 这个字段就是配置映射到外网的端口号
    non-secure-port:
     # 默认是 hostname 开头的，修改成 ip 开头
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
  client:
    serviceUrl:
      # 这里不是 localhost 了，而是注册中心的 ip:port
      defaultZone: http://ip:8080/eureka/
server:
  port: 8084
spring:
  application:
    name: services-provider    # 应用名称
```

2.**高可用注册中心**

本文上述搭建的是单个注册中心的例子，基本已经满足我目前的项目需求。但是在阅读别的博客中，发现真正的微服务架构，为了实现高可用，一般会有多个注册中心，并且相互注册形成。这里先简单做个记录。

3.**更复杂，自定义的负载均衡规则。**

目前，其实只是引入了 spring cloud 和 Eureka 的依赖就实现了简单的负载均衡。但仔细看 DynamicServerListLoadBalancer 类的位置，是在 Ribbon 下的。虽然没有显式去添加 Ribbon 的依赖包，但是实际上已经被包含进去了。那么 Ribbon 是什么？

> Ribbon is a client-side load balancer that gives you a lot of control over the behavior of HTTP and TCP clients. Feign already uses Ribbon, so, if you use @FeignClient, this section also applies.

Ribbon 是 Spring Cloud 中的一个组件，主要是做客户端负载均衡的。不关注底层细节，可能上文搭建的服务它无关，实际上还是用到了。那么，如果需要自定义复杂均衡的规则，则需要通过配置 Ribbon 来实现。

## Summary

本文主要是”拿来主义“的思想，直接用了 Spring Cloud 中的几个组件来组建一个负载均衡服务。为了能够更好地进行服务治理，还需按部就班先学习基本的原理概念，例如CAP理论等，在逐步学习 Cloud 中的组件，有一定的全局观。

注册中心 Eureka 本身满足的就是 AP，在生产环境中，为了保证服务的高可用，势必要有至少两个的注册中心。

## Reference

1. [springcloud(二)：注册中心 Eureka](http://www.ityouknow.com/springcloud/2017/05/10/springcloud-eureka.html)
2. [springcloud(三)：服务提供与调用](http://www.ityouknow.com/springcloud/2017/05/12/eureka-provider-constomer.html)
3. [Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.0.M1)
4. [SpringCloud - 负载均衡器 Ribbon](http://www.glmapper.com/2018/12/30/springcoud-ribbon-project/)
5. [Spring Cloud（四）：服务容错保护 Hystrix【Finchley 版】](https://windmt.com/2018/04/15/spring-cloud-4-hystrix/)
6. [Setup a Circuit Breaker with Hystrix, Feign Client and Spring Boot](https://blog.crafties.fr/2017/07/23/setup-a-circuit-breaker-with-hystrix/)

## [Source Code](https://github.com/maoqyhz/blog-example-code/tree/master/springboot-load-balance)
