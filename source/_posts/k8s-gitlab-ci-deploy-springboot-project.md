---
title: Gitlab CI 集成 Kubernetes 集群部署 Spring Boot 项目
tags:
  - Docker
  - Kubernetes
  - Spring Boot
  - Gitlab
  - DevOps
categories:
  - 技术
copyright: true
date: 2019-11-03 22:34:55
---

在上一篇博客中，我们成功将 Gitlab CI 部署到了 Docker 中去，成功创建了 Gitlab CI Pipline 来执行 CI/CD 任务。那么这篇文章我们更进一步，将它集成到 K8s 集群中去。这个才是我们最终的目标。众所周知，k8s 是目前最火的容器编排项目，很多公司都使用它来构建和管理自己容器集群，可以用来做机器学习训练以及 DevOps 等一系列的事情。

<!-- more -->

在这里，我们聚焦 CI/CD，针对于 Spring Boot 项目，借助 Gitlab CI 完成流水线的任务配置，最终部署到 K8s 上去。本文会详细讲解如何一步步操作，完成这样的一条流水线。

**软件的核心版本如下：**

- Kubernetes：v1.16.0-rc.2
- 部署 Gitlab 的 Docker：19.03.2
- Gitlab CE：gitlab-ce:latest
- Gitlab Runner: helm chart gitlab-runner-v0.10.0-rc1
- Helm：2.14.3

**k8s 集群信息：**

```bash
[root@master01 ~]# kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
172.17.11.62    Ready    <none>   37d   v1.16.0-rc.2
172.17.13.120   Ready    <none>   91d   v1.16.0-rc.2
172.17.13.121   Ready    <none>   91d   v1.15.1
172.17.13.122   Ready    <none>   91d   v1.16.0-rc.2
172.17.13.123   Ready    <none>   92d   v1.16.0-rc.2
```

## Runner 的安装和注册

在上一篇博客《[Docker Gitlab CI 部署 Spring Boot 项目](https://furur.xyz/2019/11/03/docker-gitlab-ci-deploy-springboot-project/)》中我们已经实现了在 Docker 中部署这一套流水线，但是单节点的 Docker 只适合本地调试，如果真正搭建起来用于公司的 CI/CD 工作，还是会把它放到集群环境下，因此现在我们将流水线部署到 k8s 上。

其实部署到 k8s 上包含两种，一个是把 Gitlab 部署上去，另一个是把 CI 这部分部署上去（也就是 Gitlab Runner）。其实 Gitlab 本身就是一个服务，部署在哪里都可以，可以选择 Docker 部署，也可以找一台服务器单独部署，作为代码仓库。最关键的其实是后者，CI/CD 的流程复杂且消耗资源多，部署在集群上就能自动调度，达到资源利用最大化。那么下面着重讲 Gitlab Runner 的 k8s 部署，想了解前者的可以看官方文档，通过 helm 安装。[GitLab cloud native Helm Chart](https://docs.gitlab.com/charts/)

假设 Gitlab 都已经部署成功了，那么下面开始安装 Gitlab Runner。具体的就是把 Runner 安装到某个节点的 pod 的上，在处理具体的 CI 任务时，Runner 会启动新的 pod 来执行任务，由 k8s 进行节点间的调度。

一般来说，我们会使用 Helm 来进行安装，Helm 是类似 CentOS 里的 yum，是一种软件管理工具，可以帮我们快速安装软件到 k8s 上。我们需要在其中一个主节点上安装好 Helm 的 client 和 server。具体可参考：

- [Helm User Guide - Helm 用户指南](https://whmzsu.github.io/helm-doc-zh-cn/quickstart/install-zh_cn.html)
- [Kubernetes Helm 初体验](https://zhuanlan.zhihu.com/p/33813367)

显示如下，证明安装成功。

```bash
[root@master01 ~]# helm version
Client: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
```

添加 gitlab 源并更新。

```bash
[root@master01 ~]# helm repo add gitlab https://charts.gitlab.io
[root@master01 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "gitlab" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[root@master01 ~]# helm search gitlab-runner
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
gitlab/gitlab-runner    0.10.0-rc1      12.4.0-rc1      GitLab Runner
```

这里看到有两个版本，一个是 chart version 一个是 app version。 chart 是 helm 中描述相关的一组 Kubernetes 资源的文件集合，里面包含了一个 value.yaml 配置文件和一系列模板（deployment.yaml、svc.yaml 等），而具体的 app 是通过需要单独去 Docker Hub 上拉取的。这两个字段分别就是描述了这两个版本号。

安装前先创建一个 gitlab 的 namespace，并为其配置相应的权限。

```bash
[root@master01 ~]# kubectl create namespace gitlab-runners
```

创建一个 `rbac-runner-config.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: gitlab-runners
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: gitlab-runners
  name: gitlab
rules:
- apiGroups: [""] #"" indicates the core API group
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: gitlab
  namespace: gitlab-runners
subjects:
- kind: ServiceAccount
  name: gitlab # Name is case sensitive
  apiGroup: ""
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: gitlab # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

然后执行以下。

```
[root@master01 ~]# kubectl create -f rbac-runner-config.yaml
```

接下来配置 runner 的注册信息。上一篇博客中，我们是先安装 gitlab runner 然后进入容器执行 gitlab-runner register 来进行注册的。在 k8s 可支持这么操作，但是同时 helm 也提供了一个配置文件可以在安装 runner 的时候为其注册一个默认的 runner。我们可以去 [gitlab-runner 的项目源码]( https://gitlab.com/gitlab-org/charts/gitlab-runner ) 中获取到 `values.yaml` 这个配置文件。配置文件比较长，可以根据需要自己去配置，下面就贴下本文中需要配置的地方。

```yaml
## 拉取策略， 先取本地镜像
imagePullPolicy: IfNotPresent

## 配置 gitlab 的 url 和注册令牌
## 可以在 gitlab 项目中设置 --CI/CD--Runner-- 手动设置 specific Runner 中查询
gitlabUrl: http://172.17.193.109:7780/
runnerRegistrationToken: "qzxposxDst_Nnq5MMnPf"

## 和之前配置的 rbac name 对应
rbac:
  serviceAccountName: gitlab

## 指定关联 runner 的标签
tags: "maven,docker,k8s"

privileged: true

serviceAccountName: gitlab
```

然后通过 helm install 安装 runner 就可以了。token 和 url 不知道如何获取的，见我上一篇博客。

```bash
[root@master01 ~]# helm install --name gitlab-runner gitlab/gitlab-runner -f values.yaml --namespace gitlab-runners
NAME:   gitlab-runner
LAST DEPLOYED: Fri Oct 25 10:08:40 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME              DATA  AGE
gitlab-runner-gitlab-runner  5     <invalid>

==> v1/Deployment
NAME              READY  UP-TO-DATE  AVAILABLE  AGE
gitlab-runner-gitlab-runner  0/1    0           0          <invalid>

==> v1/Secret
NAME              TYPE    DATA  AGE
gitlab-runner-gitlab-runner  Opaque  2     <invalid>
NOTES:
Your GitLab Runner should now be registered against the GitLab instance reachable at: "http://172.17.193.109:7780/"
```

等待一段时间后就可以在 k8s 的 dashboard 上看到启动成功的 runner 的 pod 了。

![image-20191025134309946](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/k8s-gitlab-ci-deploy-springboot-project/image-20191025134309946.png)

这个时候可以进入 pod 看一下 runner 的配置文件（`/home/gitlab-runner/.gitlab-runner/config.toml`）了。这个文件就是根据之前配置的 values.yaml 自动生成的。

```toml
listen_address = "[::]:9252"
concurrent = 10
check_interval = 30
log_level = "info"

[session_server]
  session_timeout = 1800

[[runners]]
  name = "gitlab-runner-gitlab-runner-6767fdcb6-pjvbz"
  request_concurrency = 1
  url = "http://172.17.193.109:7780/"
  token = "Soxf6HEQVj4wr25zsGUS"
  executor = "kubernetes"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
  [runners.kubernetes]
    host = ""
    bearer_token_overwrite_allowed = false
    image = "ubuntu:16.04"
    namespace = "gitlab-runners"
    namespace_overwrite_allowed = ""
    privileged = true
    service_account = "gitlab"
    service_account_overwrite_allowed = ""
    pod_annotations_overwrite_allowed = ""
    [runners.kubernetes.pod_security_context]
    [runners.kubernetes.volumes]

```

注册成功后就可以在 gitlab 的 web UI 上看到对应 token 的 runner 了。

![image-20191025145853272](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/k8s-gitlab-ci-deploy-springboot-project/image-20191025145853272.png)

## 项目的创建与 Dockerfile

这边没啥好说的，直接新建一个 spring boot 的 hello world 项目，并添加一个 dockerfile。

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY  /target/demo-0.0.1-SNAPSHOT.jar app.jar
ENV PORT 5000
EXPOSE $PORT
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-Dserver.port=${PORT}","-jar","/app.jar"]

```

## .gitlab-ci.yml 改造

由于使用的 executor 不同，所以. gitlab-ci.yml 和之前也有些不同。比如 k8s 貌似不支持缓存，volume 挂载方式的不同，以及权限问题等。先看完整的配置文件。

```yaml
image: docker:latest
variables:
  DOCKER_DRIVER: overlay2
  # k8s 挂载本地卷作为 maven 的缓存
  MAVEN_OPTS: "-Dmaven.repo.local=/home/cache/maven"
  REGISTRY: "registry.cn-hangzhou.aliyuncs.com"
  TAG: "tmp-images/hello-spring"
  TEST_POD: "hello-spring"
  TEST_POD_CONTAINER: "spring-boot"
stages:
  - package
  - build
  - deploy
  - release
maven-package:
  image: maven:3.5-jdk-8-alpine
  tags:
    - maven
  stage: package
  script:
    - mvn clean package -Dmaven.test.skip=true
  artifacts:
    paths:
      - target/*.jar
docker-build:
  tags:
    - docker
  stage: build
  script:
    - echo "Building Dockerfile-based application..."
    - docker login --username=maoqyhz@outlook.com registry.cn-hangzhou.aliyuncs.com -p [your pwd]
    - docker build -t $REGISTRY/$TAG:$CI_COMMIT_SHORT_SHA .
    - docker push $REGISTRY/$TAG
  only:
    - master
k8s-deploy:
  image: bitnami/kubectl:latest
  tags:
    - k8s
  stage: deploy
  script:
    - echo "deploy to k8s cluster..."
    - kubectl version
    - kubectl set image deployment/$TEST_POD $TEST_POD_CONTAINER=$REGISTRY/$TAG:$CI_COMMIT_SHORT_SHA --namespace gitlab-runners
  only:
    - master
#   when: manual
release:
  stage: release
  script:
    - echo "release to prod env..."
  when: manual

```

需要特别指出的地方有 3 点：

- 绑定本地 Docker 守护进程。

- maven 仓库的缓存。
- k8s 测试镜像的部署。

### 绑定本地 Docker 守护进程

Docker 环境下我们使用了一个 docker:dind 的服务用于执行 docker build 到本地镜像仓库。在 k8s 中，发现用了这个东西会出现 runner 连不到 Gitlab 服务器上，报了一个 `host xxx is unreachable` 的错误。所以最终采用 volume 绑定的形式把本地 `docker.sock` 通过 host_path 的方式挂载到 runner 中去。

```toml
# /home/gitlab-runner/.gitlab-runner/config.toml
[[runners.kubernetes.volumes.host_path]]
  name = "docker"
  mount_path = "/var/run/docker.sock"
```

### Maven 仓库的缓存

在之前的 Docker 环境下，volume 的挂载是一件很容易的事情，我们直接可以把本地的 maven 仓库挂载。或者通过 cache 节点进行缓存的配置。但是在 k8s 环境下，我折腾了很久也没找到 cache 节点的配置方法。生产环境下应该可以配置 aws 或者 minio 作为缓存。目前最终测试出来本地可行的方案是通过挂载 nfs pvc。

首先我们需要创建一个 PersistentVolume 和 PersistentVolumeClaim。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-runner-maven-repo
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  mountOptions:
  - nolock
  nfs:
    path: /opt/maven-cache/
    server: 172.17.13.120
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-maven-repo-claim
  namespace: gitlab-runners
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: gitlab-runner-maven-repo
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
```

在 runner 的配置中绑定。

```toml
# /home/gitlab-runner/.gitlab-runner/config.toml
[[runners.kubernetes.volumes.pvc]]
  name = "gitlab-runner-maven-repo-claim"
  mount_path = "/home/cache/maven"
```

然后配置 `MAVEN_OPTS` 到挂载的路径，就可以实现 maven 仓库的缓存了。

### k8s 测试镜像的部署

在 Docker 中，build 之后的镜像直接可以通过 docker run 启动。在 k8s 下相对比较复杂，需要通过配置 deployment.yaml 来进行启动，如果需要外部访问，还需要配置 services 等组件。这里仅仅是为了演示流水线的执行过程，就预先启动一个 pod 作为测试。每次触发新的构建任务时，直接通过命令 `kubectl set image` 替换掉旧镜像就可以了。

## 执行流水线

git push & commit 代码后，流水线会自动创建执行。完成后可在之前配置的 services 端口看到部署的结果。

## Troubleshooting

第一次实操过程中，坑还是很多的，在这里记录一下。

### apiVersion 发生变化产生的问题

k8s 迭代的速度较快，使用最新版但是周边生态没有跟上脚步同步更新的话很容易产生问题。在 [1.16 中 API versions 就有了比较大的变化]( https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/ )，所以导致 helm 的模板没及时更新出错了。

> Error: validation failed: unable to recognize "": no matches for kind"Deployment"in version"extensions/v1beta1"

#### 1. Helm 服务端安装失败

以往 helm 的服务端安装直接调用 helm init 就可以了。由于 apiVersion 的问题需要通过以下命令才能装上。完整的安装的命令如下：

```bash
# 添加权限控制账户
[root@master01 ~]# kubectl create serviceaccount --namespace kube-system tiller
[root@master01 ~]# kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

# 通过此命令修改 apiVersion 由 extensions/v1beta1 变为 apps/v1，并使用国内镜像。
[root@master01 ~]# helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3 --service-account tiller --override spec.selector.matchLabels.'name'='tiller',spec.selector.matchLabels.'app'='helm' --output yaml | sed 's@apiVersion: extensions/v1beta1@apiVersion: apps/v1@' | kubectl apply -f -

[root@master01 ~]# [root@master01 ~]# helm repo remove stable
[root@master01 ~]# helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

#### 2. Gitlab 无法直接托管 k8s 集群安装 Runner

同样的，上文我们是通过手动安装的 helm server 和 gitlab runner。而实际上 Gitlab CE 支持免费绑定 1 个 k8s 集群，并且提供一键式的安装。但是由于版本号的问题，我这里安装不上。大家可以用低版本的 k8s 测试一下。具体在 Gitlab WebUI 中 root 账户登录，管理中心 -- Kubernetes 或者项目管理界面下，运维 --Kubernetes 进行集群的绑定。

![image-20191025170006350](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/k8s-gitlab-ci-deploy-springboot-project/image-20191025170006350.png)

绑定后会托管到 Gitlab，创建一个 namespace 来进行程序的安装。

![image-20191025170129330](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/k8s-gitlab-ci-deploy-springboot-project/image-20191025170129330.png)

### Runner 中 config.toml 配置消失

之前在配置 runner 的 volume 的时候是直接进入 pod 内部修改 config.toml 这个配置文件的。这样会存在一个问题，如果集群重启了，修改过的数据就全没了，因为重启后会重新起一个 pod。以往在 Docker 中，我们会 commit 成一个新的镜像保存，但这样保存的镜像显然没有普遍性，并且每次还需要修改 deployment.yaml 中的镜像，因此最好的方法是修改 helm 的 chart 中的模板，把所有的配置信息都写入 yaml 文件中，而非直接修改 pod 本身中的配置。

因此为了能够高度的配置我们自己的 runner，就不直接从 helm 官方的源镜像安装了。

```bash
[root@master01 ~]# helm fetch gitlab/gitlab-runner
```

helm fetch 可以将 chart 下载到本地，可以看到是. tgz 格式的，解压后的文件夹内包含的就是描述资源的一些模板文件。

```
gitlab-runner-v0.10.0-rc1
├── CHANGELOG.md
├── Chart.yaml
├── CONTRIBUTING.md
├── LICENSE
├── NOTICE
├── README.md
├── scripts
│   ├── changelog2releasepost
│   └── prepare-changelog-entries.rb
├── templates
│   ├── _cache.tpl
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── _env_vars.tpl
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── NOTES.txt
│   ├── role-binding.yaml
│   ├── role.yaml
│   ├── secrets.yaml
│   └── service-account.yaml
└── values.yaml
```

之前提取的 values.yaml 和这里是一样的，用于设置一些可配置的变量。具体的模板在 templates 目录下。这里我们可以仔细看下 values.yaml、deployment.yaml 和 configmap.yaml，会发现 runner 的 config.toml 文件中的大部分信息都是根据这三个配置文件中的信息自动生成的。那么我们只需要额外配置一下需要的 volume 就可以了。

这里我们需要修改 vaules.yaml 和 configmap.yaml。

```yaml
## configmap.yaml
if ! sh /scripts/register-the-runner; then
  exit 1
fi

# add volume config
cat >>/home/gitlab-runner/.gitlab-runner/config.toml <<EOF
  [[runners.kubernetes.volumes.pvc]]
	name = "{{.Values.maven.cache.pvcName}}"
	mount_path = "{{.Values.maven.cache.mountPath}}"
  [[runners.kubernetes.volumes.host_path]]
	name = "docker"
	mount_path = "/var/run/docker.sock"
EOF

# Start the runner
exec /entrypoint run --user=gitlab-runner \
  --working-directory=/home/gitlab-runner
```

在 register 和 start 之间添加配置 volume 的信息，并在 values.yaml 配置相应的变量。

```yaml
## my config
#  maven cache
maven:
  cache:
    pvcName: gitlab-runner-maven-repo-claim
    mountPath: /home/cache/maven
```

配置完成后，执行 `helm upgrade gitlab-runner gitlab-runner-v0.10.0-rc1/`。

之后无论重启 pod 还是集群，runner 的配置信息都能被正确加载了。

### 镜像下载不到的问题

gcr.io 无法访问，Docker Hub 访问速度慢，这个大家都懂的。

- 阿里云镜像加速器。
- 寻找能用的镜像 pull 下来本地打 tag。[七牛云镜像中心]( https://hub.qiniu.com/home#page=web )
- 寻找本地的镜像服务器

阿里云通用镜像服务器： https://registry.cn-hangzhou.aliyuncs.com

helm chart 镜像

- 阿里：https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
- Azure：http://mirror.azure.cn/kubernetes/charts

## 总结

本文主要描述了在 k8s 上部署流水线的整个过程，由于对 k8s 不太熟悉，遇到了不少的坑，国内相关的博客也较少，所以就记录一下整个配置的过程。对于整个流程不熟悉的读者可以先阅读上一篇博客《[Docker Gitlab CI 部署 Spring Boot 项目](https://furur.xyz/2019/11/03/docker-gitlab-ci-deploy-springboot-project/)》。下一步的工作就是要将流水线对接到实际的项目中去了。

未完待续...
## [源码](https://github.com/maoqyhz/tequila/tree/master/springboot-gitlab-ci-k8s)

## 参考资料

- [GitLab Runner Helm Chart]( https://docs.gitlab.com/runner/install/kubernetes.html )
- [GitLab CI/CD Pipeline Configuration Reference]( https://docs.gitlab.com/ee/ci/yaml/ )
- [Helm User Guide - Helm 用户指南]( https://whmzsu.github.io/helm-doc-zh-cn/ )
- [使用 GitLab CI 在 Kubernetes 服务上运行 GitLab Runner 并执行 Pipeline]( https://help.aliyun.com/document_detail/106968.html )