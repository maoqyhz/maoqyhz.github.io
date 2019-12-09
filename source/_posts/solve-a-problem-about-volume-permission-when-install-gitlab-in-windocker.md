---
title: 解决 Windows Docker 安装 Gitlab Volume 权限问题
tags:
  - Docker
  - Gitlab
categories:
  - 技术
copyright: true
date: 2019-09-21 14:01:43
---

记录一下 Windows10 下 Docker 安装 Gitlab 的步骤。

<!-- more -->

> **Caution:** We do not officially support running on Docker for Windows. There are known issues with volume permissions, and potentially other unknown issues. If you are trying to run on Docker for Windows, please see our [getting help page](https://about.gitlab.com/get-help/) for links to community resources (IRC, forum, etc) to seek help from other users.

首先，**Gitlab 官方是不支持 Windows 下部署 Gitlab 镜像的，所以正常的 Gitlab 服务还是部署在 Linux 上比较好。本地部署只是用于个人开发测试环境。**


## 问题描述

其实搭建 Gitlab 本省是一件很简单的事情，直接 pull 官方的 Gitlab 镜像开起来就可以用了。

```
docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest
```

在 Windows 下我们把 volume 配置成本地路径运行后会出现一下错误：

```
Error executing action create on resource 'storage_directory[/var/opt/gitlab/git-data]
```

通过查找，这应该是权限不足，导致 Windows 下的 volume 映射存在一些问题。

## 解决方法

别人探索出目前可用的方法是**采用 volume 数据卷挂载的形式。**

首先先安装 Docker for Windows。并在 Setting 中设置 Shared Drives，设置一会用于挂载 docker 镜像的 volume 的磁盘。

然后初始化配置文件路径和 volume。

```bash
mkdir D:\docker\gitlab\config
mkdir D:\docker\gitlab\backups
docker volume create gitlab-logs
docker volume create gitlab-data
```

然后直接创建一个 Container 运行就可以了。

```bash
docker run --detach `
    --name gitlab `
    --restart always `
    --hostname localhost `
    --publish 10443:443 --publish 10080:80 --publish 1022:22 `
    --volume D:\docker\gitlab\config:/etc/gitlab `
    --volume gitlab-logs:/var/log/gitlab `
    --volume gitlab-data:/var/opt/gitlab `
    gitlab/gitlab-ce
```

等待一段时间初始化后，就可以访问本地的 10080 端口了，http://localhost:10080

打开后就是正常 Gitlab 的页面，重置一下 root 的密码就可以正常使用了。

### 使用 Docker-Compose 部署（推荐）

如果在运行 Docker 容器时需要配置很多的参数，显然一遍遍输入 `docker run` 会比较麻烦，这里可以采用三剑客当中的 Docker-Compose 来进行容器的管理和创建（安装 docker-ce 时默认安装）。暂时不管 Docker-Compose 的其他用法，其实就是把命令运行改成了文件运行而已。

Docker-Compose 是通过文件来创建 Docker Container 的。我们需要在一个目录下创建 `docker-compose.yml` 文件，写入相应的配置文件。现在我们把上面的命令进行改造：

```yaml
# Compose file 版本号，和 docker 版本号对应。3 支持 docker 1.13.0+
version: "3"
# services 节点下包含多个待创建的 Docker Container
services:
  # web 节点就是待启动的 gitlab 容器
  web:
    image: gitlab/gitlab-ce:latest
    container_name: "gitlab"
    restart: always
    hostname: localhost:10080
    environment:
      TZ: "Asia/Shanghai"
      GITLAB_OMNIBUS_CONFIG: |
        gitlab_rails["time_zone"] = "Asia/Shanghai"
        gitlab_rails["gitlab_shell_ssh_port"] = 10022
        nginx["listen_port"] = 80
    ports:
      - "10080:80"
      - "10022:22"
    volumes:
      - D:\docker\gitlab\config:/etc/gitlab
      - gitlab-logs:/var/log/gitlab
      - gitlab-data:/var/opt/gitlab
volumes:
  gitlab-logs:
  gitlab-data:
```

可以看到这个文件的内容几乎和之前的 `docker run` 命令是保持一致的，唯一不同的是不需要我们自己创建 volume 了，直接在配置文件中配置后，启动时会自己为我们创建。

配置完成后，使用 docker-compose 命令运行起来。

```
# 打开 cmd，进入 docker-compose.yml 的根目录
# 创建容器
docker-compose up -d

#关闭容器
docker-compose stop
```

## What's More

### 1. Web UI 端口显示问题

由于 Gitlab 是在 Docker 内运行的，外部需要访问的话都是需要通过端口映射的，并且一般内部端口不会和映射出来的外部端口相同。所以在用的时候可能会出现一些问题。

例如在我们例子里，22 映射到 10022,80 映射到 10080。可以看到在 Gitlab 默认的 WebUI 中，项目显示的克隆地址默认是不带端口号的,如下图所示：

![](https://img2018.cnblogs.com/blog/697102/201909/697102-20190924165751351-398741507.png)

因此在进行克隆的时候，无论是 http 还是 ssh，都需要在 url 中手动添加新的端口，例如

```ruby
http://localhost:10080/root/demo.git
```

修改配置文件后可以直接在 WebUI 中显示正确的 url。

具体需要修改 `gitlab.rb` 和容器内部 `/var/opt/gitlab/gitlab-rails/etc/gitlab.yml`。

首先修改 `gitlab.rb`

```yaml
# 取消这条配置文件的注释，并修改为外部映射的 ssh 端口
gitlab_rails['gitlab_shell_ssh_port'] = 1022
# 使用 exec 进入容器内部
root@gitlab:/# gitlab-ctl reconfigure
```

再修改 `/var/opt/gitlab/gitlab-rails/etc/gitlab.yml`

```
gitlab:
  ## Web server settings (note: host is the FQDN, do not include http://)
  host: 127.0.0.1
  port: 10080 # 修改此处
  https: false
# 修改完后执行
root@gitlab:/# gitlab-ctl stop
root@gitlab:/# gitlab-ctl start
```

**这里要注意，后面的那个配置文件是由前面那个生成的，修改 gitlab.rb 后 reconfigure，后面那个配置文件就会被重置了，注意一下修改的顺序。**

显然这种方法比较麻烦，且需要进入容器中修改，一旦重启就没了。如果使用 docker-compose 来启动容器的话，可以直接在 environment 的 GITLAB_OMNIBUS_CONFIG 节点中配置，具体配置方式看上面的配置文件。

### 2. 镜像备份问题

由于使用的是 volume，因此 gitlab 内部的数据直接由 docker 管理了。显然就不太友好。如果有这个需求的可以阅读参考文献 2，里面提到了备份的方法。

## 总结

总之，Windows 对 Docker 的支持不是很友好，除了下一个学习学习，尝尝鲜，或者用于安装一些 Windows 下无法安装的软件，例如 Redis 等外，并不建议使用，显然选择 linux 系统一个是更明智的选择。

## Reference

1. [GitLab Docker images](https://docs.gitlab.com/omnibus/docker/)
2. [Volume trouble with GitLab docker image on Windows](https://stackoverflow.com/questions/44684621/volume-trouble-with-gitlab-docker-image-on-windows)
3. [I can not run Gitlab-ce on Docker via Windows if I set "volume" to Windows Shared Drives](https://gitlab.com/gitlab-org/omnibus-gitlab/issues/3443)
4. [Gitlab 备份和迁移](https://zhuanlan.zhihu.com/p/56108334)