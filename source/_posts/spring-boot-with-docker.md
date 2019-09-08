---
title: Windows Docker 部署 Spring Boot 项目
tags:
  - Spring Boot
  - Docker
categories:
  - 技术
copyright: true
date: 2019-08-28 22:32:08
---

使用 Docker 部署一个简单的 Spring Boot 数据库项目。

<!--more-->

最近容器化技术 hin 流行啊，所以开始折腾一下呗。试用了下，有的时候的确比虚拟机要方便。其实起初是要用 Redis，但是 Windows 安装不方便，于是就看了下 Docker，基本开箱即用，Docker Hub 上 pull 一个 Redis 官方镜像就可以了。刚好最近在学 Spring Cloud 微服务，也就尝试下 Docker 部署 Spring Boot 项目。

内容上，本文主要是对 Spring Boot 官方文档的一个补充，英语不错的可以直接浏览参考文献 6-7。

Docker 的基本概念和用法，这里就详细展开了，可以看参考文献 1-3。

## Docker Configuration

Docker 一般是安装在 Linux 系统中的，但是目前 Windows 也能使用，下载安装后开箱即用。额外需要配置的是：

1. 暴露 TCP 端口，可以在 IDEA 的 Docker 插件中连接使用。

   ![1566891262146](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566891262146.png)

2. 替换官方的 Docker 仓库，使用阿里云镜像加速。在 `Daemon` 中可进行配置。

### Config IDEA Plugin

Docker 本身是没有 GUI 界面的，所以所有的维护管理都必须使用命令行，比较麻烦，而 IDEA 给我们提供了一个简单的 Docker 插件，用于管理 Image 和 Container。

![配置docker插件](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566893567141.png)

在设置中直接搜索 docker，添加一个 Docker Engine，配置 URL，稍等下面提示连接成功后，就可以了。如果连接失败了，可以**检查下 TCP 的 2375 端口是否暴露**了。

![插件dashboard](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566893744915.png)

然后在底部，就能看到 Docker 的 tab了。里面的信息和命令行中看到的一致。基本的创建、删除、端口映射等操作都能在上面进行。

那么 Docker 相关的配置就完成了。

## Create Spring Boot Project

这里用来演示的是一个 Spring Boot 的数据库项目。跟之前一样，配置一些基本内容就可以了。

```xml
<!--pom.xml-->
<groupId>com.example</groupId>
<artifactId>spring-docker</artifactId>
<version>1.0.0</version>
<name>spring-docker</name>

<properties>
  <java.version>1.8</java.version>
  <skipTests>true</skipTests>		
  <druid-spring-boot-starter.version>1.1.10</druid-spring-boot-starter.version>
  <dockerfile-maven-plugin.version>1.4.10</dockerfile-maven-plugin.version>
  <docker.image.prefix>springboot</docker.image.prefix>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>${druid-spring-boot-starter.version}</version>
  </dependency>
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

```yaml
# application.yml
# 配置数据库信息
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://ip:port/test?serverTimezone=Hongkong&characterEncoding=utf-8&useSSL=false
    username: 
    password: 
```

```java
// Spring Boot 启动类
@SpringBootApplication
@RestController
public class SpringDockerApplication {

    private final JdbcTemplate jdbcTemplate;
    public SpringDockerApplication(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @GetMapping("/")
    public Message index() {
        RowMapper<Message> rowMapper= new BeanPropertyRowMapper<>(Message.class);
        Message msg = jdbcTemplate.queryForObject("select * from message where k = ?", rowMapper,"msg");
        return msg;
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringDockerApplication.class, args);
    }
}
```

本地访问，能成功运行就说明，Spring Boot 已经配置完成了。下面就是容器化的操作了。

## Containerize It

容器化其实就是创建 Docker 镜像的过程，需要把 jar 包封装进去。具体有两种方式：

1. 创建 Dockerfile，手动执行 build 和 run。
2. 使用 dockerfile-maven-plugin 插件。

### Use Dockerfile

首先，在 `src/main ` 下创建 `docker` 文件夹，并创建一个 Dockerfile，内容如下：

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY spring-docker-1.0.0.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

假设大家对 Dockerfile 有一定的了解，解释一下上面的内容，

- FROM...，表示使用 `openjdk` 作为基本镜像，标签为：8-jdk-alpine，意思 jdk 版本为 1.8，精简版。
- VOLUME...，创建一个 `VOLUME` 指向 `/tmp` 的内容。因为这是 Spring Boot 应用程序默认为 Tomcat 创建工作目录的地方。效果是在 `/ var / lib / docker` 下的主机上创建一个临时文件，并将其链接到 `/tmp` 下的容器。
- COPY...，把项目中的 jar 文件复制过去，并重命名为 `app.jar`。
- ENTRYPOINT...，执行 `java -jar` 启动项目。`java.security.egd=file:/dev/./urandom` 的目的是为了缩短 Tomcat 启动的时间。

接下来，使用 maven 打包一个 jar，手动复制到 `src/main/docker`目录下。

配置 IDEA 来 build 和 run docker。

![配置docker run](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566898104800.png)

需要配置的参数如下：

![具体配置流程](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566898348067.png)

1. Name，创建的配置文件名称。
2. Image tag，创建镜像的 repository:tags。
3. Container name，创建容器名。
4. Bind posts，端口映射，这里将默认的 8080 端口映射到外网的 8888 端口。
5. 最后 IDEA 会将参数拼接成 docker build & run 的命令。

点击 Run。镜像就会被自动 build，并且在容器中运行了。

![查看部署日志](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566901917949.png)

选择我们刚刚部署的那个容器的 Log tab，就能看到 Spring Boot 启动的日志了，是不是有一种似曾相识的感觉 :smiley:

![spring boot日志](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566902052318.png)

假设之前我们连接的是本地的 MySQL那么执行到这里应该会报错了，错误如下：

```
2019-08-27 10:37:04.197 ERROR 1 --- [eate-1939990953] com.alibaba.druid.pool.DruidDataSource   : create connection SQLException, url: jdbc:mysql://127.0.0.1:3306/test?serverTimezone=Hongkong&characterEncoding=utf-8&useSSL=false, errorCode 0, state 08S01

com.mysql.cj.jdbc.exceptions.CommunicationsException: Communications link failure
Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure
Caused by: java.net.ConnectException: Connection refused (Connection refused)
```

这是由于我们配置数据库 url 出现了问题，本地数据库，一般会写 `localhost` 或 `127.0.0.0.1`。但是这个地址在 Docker 容器中是不认的。所以连不上本地数据库。幸好 Docker 在安装的时候配置了一个虚拟网桥，通过那个地址可以和 Docker 容器内部进行通信。

我们在宿主机上的 cmd 中输入 `ipconfig` ，就可以看到。

```
C:\Users\mqy6289>ipconfig
Windows IP 配置

以太网适配器 vEthernet (DockerNAT):
   连接特定的 DNS 后缀 . . . . . . . :
   IPv4 地址 . . . . . . . . . . . . : 10.0.75.1
   子网掩码  . . . . . . . . . . . . : 255.255.255.240
   默认网关. . . . . . . . . . . . . :

以太网适配器 以太网:
   连接特定的 DNS 后缀 . . . . . . . : apac.arcsoft.corp
   本地链接 IPv6 地址. . . . . . . . : fe80::c64:6e72:8311:378c%13
   IPv4 地址 . . . . . . . . . . . . : 172.17.218.99
   子网掩码  . . . . . . . . . . . . : 255.255.224.0
   默认网关. . . . . . . . . . . . . : 172.17.192.1
```

在 DockerNAT 中，宿主机的 ip 为 10.0.75.1，将数据库的 url ip 改成这个。将之前的容器和镜像删除后，重新执行之前的步骤，发现部署成功。

在浏览器中输入：http://127.0.0.1:8888/ ，就能看到 hello world 了。整个 Docker 部署的过程完成。

![浏览结果](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566902991835.png)

针对之前的网络通信问题，我们可以在 cmd 中输入 `docker inspect hello-world` 来查看容器的参数。可以看到容器的 IP 和网关的确不在一个网段，无法进行通信。

```json
{
    "Networks": {
        "bridge": {
            "Gateway": "172.17.0.1",
            "IPAddress": "172.17.0.2",
            "MacAddress": "02:42:ac:11:00:02",
        }
    }
}
```

### Use Maven Plugin

之前我们创建完 `dockerfile` 之后通过 IDEA 进行镜像的 build 和 run 工作，同样的我们也可以通过 Maven 插件将 Docker 镜像和容器的构建和 Maven 集成起来。阅读了很多博客，大多数任然使用 `docker-maven-plugin`，但是这个插件早就已经被官方废弃，不推荐使用，**[spotify](https://github.com/spotify) 也推荐使用 `dockerfile-maven-plugin` 来代替**。下面的 pom.xml 的配置：

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>dockerfile-maven-plugin</artifactId>
  <version>${dockerfile-maven-plugin.version}</version>
  <configuration>
    <contextDirectory>src/main/docker</contextDirectory>
    <repository>${docker.image.prefix}/${project.artifactId}</repository>
    <tag>v1</tag>
  </configuration>
</plugin>
```

刷新后，可在 IDEA 的 Maven tab 看到插件的功能，主要用到的就是 build、push、tag。

默认插件会找本地 `tcp://localhost:2375` 如果开启了，双击 `dockerfile:build` 镜像就会被 build 到 Docker 中去。

![maven部署日志](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/spring-boot-with-docker/1566956922463.png)

可以看到，整个 build 的过程被放到 Maven 中去了。需要运行的话，选择相应的镜像，`create container` 再做简单配置就可以了。

## Troubleshooting

1. **数据库连接不上。**

本地数据库需要配置支持远程额访问，由于是 Windows，一般不会配置，可能会忽视。

```sql
UPDATE USER SET HOST = '%' WHERE USER = 'root';
FLUSH PRIVILEGES;
```

2. **进入终端**

我们这个基础镜像是 Java，默认应该是不带终端的，如果要查看的话，可再起一个容器

```
docker run -ti --entrypoint /bin/sh springboot/spring-docker:v1
```

进入后输入 `ls -al`，可以看到之前复制过去的 `app.jar` 文件

## Summary

本文主要讲了如果在 Docker 中部署 Spring Boot 项目，针对于 Dockfile 的写法和 Maven 插件的使用，可以看参考文献 5-6，里面有更多的细节。我个人觉得这种部署方法，直接把 jar 和基本的环境打成一个 Docker 镜像还是过于简单了，很难去做监控和管理。是否还是应该像服务器部署那样，创建一个基本的 Centos 镜像，然后安装 Java 的环境，通过 ssh 连接，直接把 jar 包和配置文件复制到容器中通过 bash 运行，这种方式可能更符合常理。后期如果工作上有需求，可以再深度学习一下，看看实际生产中是如何操作的。

## Reference

1. [Docker Get Started](https://docs.docker.com/get-started/)
2. [30 分钟快速入门 Docker 教程](https://juejin.im/post/5cacbfd7e51d456e8833390c#heading-13)
3. [Docker(一)：Docker入门教程](https://www.cnblogs.com/ityouknow/p/8520296.html)
4. [Spring Boot 2.0(四)：使用 Docker 部署 Spring Boot](https://www.cnblogs.com/ityouknow/p/8599093.html)
5. [Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker/)
6. [Spring Boot Docker](https://spring.io/guides/topicals/spring-boot-docker)

## [Source Code](https://github.com/maoqyhz/blog-example-code/tree/master/spring-docker)