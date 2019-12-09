---
title: Docker Gitlab CI 部署 Spring Boot 项目
tags:
  - Docker
  - Spring Boot
  - Gitlab
  - DevOps
categories:
  - 技术
copyright: true
date: 2019-11-03 22:31:01
---

目前在学习这一块的内容，但是可能每个人环境都不同，导致找不到一篇博客能够完全操作下来没有错误的，所以自己也写一下，记录一下整个搭建的过程。

<!-- more -->

Docker 的安装这里就不赘述了，基本上几行命令都可以了，不会的可以搜一下其他的博客。我本地使用的环境如下：

- Ubuntu16.04
- Docker19.03
- 管理工具：IDEA Docker 插件

下面详细讲一下部署的过程。

## 前言

阅读这篇博客的朋友关注点应该在 Gitlab CI 上，因此假设大家对 Docker 和 Gitlab 本身是有一定的了解，掌握基本使用的。

对于 CI/CD 以及 Gitlab CI，这里也没打算展开讲，本文的目的在于实战，如果想对 CI/CD 概念以及 Gitlab CI 当中的内容有兴趣的可以阅读参考文献 1-2。

## 安装 Gitlab CE 和 Gitlab Runner

这里推荐在 Linux 系统下学习 Docker，随着镜像和容器在使用上的复杂性越来越来，Win 下出现的坑会越来越多的。Gitlab CE 的搭建很简单，直接使用官方的镜像 docker run 就可以了。但是在我们这里由于还需要部署 Runner，多个容器的管理，使用 Docker-Compose 会更好，因此这里采用它来进行。

可参考我之前写的文章，[解决 Windows Docker 安装 Gitlab Volume 权限问题](https://furur.xyz/2019/09/21/solve-a-problem-about-volume-permission-when-install-gitlab-in-windocker/)，里面也提到了如何用 Docker-Compose 进行安装。这里我就直接给出配置文件了。

```yaml
version: '3'  #1
services:
    gitlab:
      image: gitlab/gitlab-ce:latest  #2
      container_name: "gitlab"
      restart: unless-stopped
      privileged: true
      hostname: "172.17.193.109:7780"  #3
      environment:
        #4
        GITLAB_OMNIBUS_CONFIG: |
          # external_url 'http://172.17.193.109:7780'
          gitlab_rails["time_zone"] = "Asia/Shanghai"
          gitlab_rails["gitlab_shell_ssh_port"] = 7722
          nginx["listen_port"] = 80
      #5
      ports:
        - "7780:80"
        - "7722:22"
      #6
      volumes:
        - /home/cache/gitlab/config:/etc/gitlab
        - /home/cache/gitlab/data:/var/opt/gitlab
        - /home/cache/gitlab/logs:/var/log/gitlab
    gitlab-runner:
      container_name: gitlab-runner
      image: gitlab/gitlab-runner:latest  #7
      restart: always
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock"
        - "/home/cache/gitlab/runner-config:/etc/gitlab-runner"
```

具体说明如下（docker-compose.yml 的文件骨架这里不做解释）：

- #1：为 docker-compose 文件的版本号，它和 docker 版本有一个对应，具体在 docker 官方文档中有明确说明。一般写 3 就可以了。
- #2：当前部署使用 Gitlab 官方最新版的镜像。
- #3：hostname 填写的是部署成功后，Gitlab 的入口地址。由于我使用的 docker 不在本地，所以配置了一个 ip 地址，本地写 localhost 就可以了。当前配置的意思为，通过 172.17.193.109 的 7780 端口访问 Gitlab 页面。
- #4：GITLAB_OMNIBUS_CONFIG 为 Gitlab 的配置参数，对应 `/etc/gitlab/gitlab.rb`。该文件下的所有键值对都可以在这里进行配置，容器启动时会自动配置进去。当然也可以在 Gitlab 容器启动后，手动修改 gitlab.rb 文件。
- #5：配置了需要用到的两个端口。默认 Gitlab 容器内部 80 端口用于 Gitlab 页面的访问，22 用于 ssh 连接远程仓库。分别对其进行外网映射。
- #6：volume 映射，三个 volume 和官方文档一致就可以了。
- #7：这里用的是 gitlab-runner 镜像，部分博客使用的 gitlab-ci-multi-runner 是旧版本，最新的镜像统一修改为 gitlab-runner。

接下来直接在 docker-compose.yml 的根目录运行就 ok 了。

```
docker-compose up -d
```

这个时候可以看 IDEA 的 Docker 插件。

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569321865259.png)

这样就说明容器正在初始化，等待一会，打开之前配置好的 ip:port，就能看到 Gitlab 页面了。首次使用需要重置 root 账户密码。接下去就是正常的使用。gitlab-runner 先不管，后面会讲到他，目前不需要做任何的配置工作, 只要正常启动即可。

## 创建一个 Spring Boot 项目

接下来我们使用 Gitlab CI 构建的项目时一个基于 Spring Boot 的 hello world 项目，我们先把他创建出来。为了简化这个过程，我们直接在新建项目的时候选择 Spring Boot 模板，它会为我们生成一个 hello world 项目，并且包含了一个 Dockerfile。

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569321452742.png)

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569321599383.png)

项目结构上和我们手动创建的是一样的。那么这个时候准备工作基本上就做完了。在进入 Gitlab CI 的流程前，我们可以想象一下，在 Spring Boot 项目部署的过程中，有哪些步骤是可以让 Gitlab CI 来完成的。我们最终的目的是，希望通过 Gitlab CI，直接可以将我们 push 到远端仓库的代码自动构建，并在一个新的容器中运行。那么具体的步骤应该有 2 步：

1. 拉取最新的代码，打包成 jar。
2. 将 jar 容器化，进一步构建成为 Docker 镜像并运行起来。

那么接下来的操作就是围绕着这两步展开的。

## 配置 Runner

安装完 Gitlab-Runner 并且创建好项目后，就需要为我们的项目注册具体的 runner 来执行 CI 任务。

首先我们打开，Gitlab 项目的设置 --->CI/CD--->Auto DevOps。看到 URL 和注册令牌。

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569323287879.png)



进入 gitlab-runner 容器内部（exec /bin/bash），执行 gitlab-runner register 开始注册。

```bash
root@bb6040f1cd04:/# gitlab-runner register
Runtime platform                                    arch=amd64 os=linux pid=43 revision=a987417a version=12.2.0
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://172.17.193.109:7780/
Please enter the gitlab-ci token for this runner:
zpzZ-shsCVxsJDZtAPNZ
Please enter the gitlab-ci description for this runner:
[bb6040f1cd04]: hello,spring boot!
Please enter the gitlab-ci tags for this runner (comma separated):
maven,docker
Registering runner... succeeded                     runner=zpzZ-shs
Please enter the executor: docker-ssh, shell, ssh, virtualbox, docker+machine, custom, parallels, docker-ssh+machine, kubernetes, docker:
docker
Please enter the default Docker image (e.g. ruby:2.6):
docker:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

根据步骤，依次输入对应的值，显示注册成功则完成注册。这个时候刷新 Gitlab 页面，可以看到刚刚注册成功的 runner。

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569323661605.png)

由于刚刚我们只是根据流程配置了一些基本的信息，还有额外的参数要配置就需要修改对应的配置文件了。可以直接修改映射到本地的 `/home/cache/gitlab/runner-config` 目录下的 `config.toml`。每配置一个 runner 就会在配置文件中生成一个 [[runners]]。

```toml
[[runners]]
  name = "hello,spring boot!"
  url = "http://172.17.193.109:7780/"
  token = "u8_Y5rLQazUmBZar9eys"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "docker:latest"
    privileged = true  #1
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/home/cg/.m2:/root/.m2"]  #2
    shm_size = 0
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

需要修改的地方：

- #1：默认为 false，需改为 true。false 时，在 CI 构建的时候 会进行 health check，很耗时而且还是失败，设为 true 就自动跳过了，其中原因暂未深究。
- #2：如果本地有 maven 环境的话，可以挂在到本地，这样在处理依赖时可以直接使用本地的环境，并且可以用阿里云镜像源。

那么 runner 是需要触发才能工作的，接下来就需要配置 Gitlab-CI 了 。

## 容器化与 Gitlab CI Auto DevOps 配置

容器化就是 jar 转化为 Docker 镜像的过程。我们讲之前自动生成的 Dockerfile 修改一下：

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY  /target/demo-0.0.1-SNAPSHOT.jar app.jar
ENV PORT 5000
EXPOSE $PORT
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-Dserver.port=${PORT}","-jar","/app.jar"]
```

然后添加 Gitlab CI 核心的配置文件——`.gitlab-ci.yml`，并把它放在项目的根目录下。Gitlab 项目在创建的时候，默认会开启 Auto DevOps流水线，当有代码 push 到仓库中去的时候会自动扫描根目录下是否包含 `.gitlab-ci.yml`，如果有，会根据预定义的持续集成和持续交付配置自动化地构建、测试和部署应用程序。那么下面来看具体的配置文件：

```yaml
image: docker:latest  #1
variables:  #2
  DOCKER_DRIVER: overlay2
  DOCKER_HOST: tcp://172.17.193.109:2375  # docker host，本地可不写
  TAG: root/hello-spring:v0.1  # 镜像名称
cache:  #3
  paths:
    - .m2/repository
services:  #4
  - docker:dind
stages:  #5
  - package
  - deploy
maven-package:  #6
  image: maven:3.5-jdk-8-alpine
  tags:
    - maven
  stage: package
  script:
    - mvn clean package -Dmaven.test.skip=true
  artifacts:
    paths:
      - target/*.jar
build-master:  #7
  tags:
    - docker
  stage: deploy
  script:
    - docker build -t $TAG .
    - docker rm -f test || true
    - docker run -d --name test -p 5000:5000 $TAG
  only:
    - master
```

具体说明：

- #1：需要用到的镜像
- #2：必须配置的一些环境变量。如果本地可不配置 `DOCKER_HOST`。
- #3：配置缓存，配置后，maven 下载的依赖可以被缓存起来，下次不需要重复去下载了。
- #4：配置需要用到的额外的服务。docker:dind，这个貌似是用于在 docker 中运行 docker 的一种东西，在项目的构建中需要。
- #5：stages，这是 Gitlab CI 中的概念，Stages 表示构建阶段，就是一些按序执行的流程，具体执行是依赖于 Jobs 的。
- #6 ：定义的 Jobs 之一，用于构建 jar 包。内部又引入 maven 镜像来处理，负责执行 package 这一流程。script 为具体执行的脚本。
- #7：定义的 Jobs 之一，用于构建 Docker 镜像。负责执行 deploy 这一流程。具体执行 build 和 run。only 节点表示只监控 master 分支。

好了，所有的配置工作都已经完成了，接下来 Git 执行 commit&push，自动构建就开始了。回到 Gitlab 页面就可以看到构建的过程。

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569373765557.png)

可以看到，目前的 Auto DevOps 中的流水线已经触发了，总共会一次执行两个阶段（package、deploy）。分阶段可以查看日志。

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569373899525.png)

![]( https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/docker-gitlab-ci-deploy-springboot-project/1569373925520.png)

两个阶段都完成后，可以直接在浏览器中访问之前配置的 5000 端口，就能看见 hello，spring 的页面了。

## 总结

本文讲了在 Docker 中部署 Gitlab，并尝试使用它的 CI 功能。Gitlab CI 这类工具对于测试和生产部署还是很有意义的，无论是在规范性还是便捷性上。而 Docker 提供了一个便捷的部署环境，尤其是像目前我司这样需要跨平台调用的场景。

## References

1. [什么是 CI/CD？](https://www.redhat.com/zh/topics/devops/what-is-ci-cd)
2. [用 GitLab CI 进行持续集成](https://scarletsky.github.io/2016/07/29/use-gitlab-ci-for-continuous-integration/)
3. [Continuous delivery of a Spring Boot application with GitLab CI and Kubernetes](https://about.gitlab.com/2016/12/14/continuous-delivery-of-a-spring-boot-application-with-gitlab-ci-and-kubernetes/)
4. [GitLab-CI 环境搭建与 SpringBoot 项目 CI 配置总结](https://blog.yupaits.com/ops-deploy/gitlab-ci-springboot.html)