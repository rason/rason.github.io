---
title: 将应用从 SpringCloud 迁移到 k8s
date: 2019-11-27 10:41:56
tags:
---

最近花了几天时间看了一下 k8s 和 istio 的文档，对容器化运维以及服务网格有了基础的了解。俗话说读万卷书不如行万里路，于是先尝试用 minikube 练一下手，将现有了一个 Spring Cloud 项目迁移到 k8s 上来。

粗略地整理了一个整个流程，主要有以下几个改动点：

1. 代码与配置修改
2. 编写 Dockerfile
3. 安装 kubectl 和 minikube
4. 创建 configmap 资源
5. 创建 deployment 资源
6. 创建 secret 资源
7. 暴露服务给集群外部调用

### 代码与配置修改

代码与配置的修改点不多，这样充分说明从 SpringCloud 迁移到 k8s 是相当的友好。

1. 由于 k8s 有内置 dns 做服务地址解析以及 service 资源做负载均衡。所以我们不再需要 eureka 做服务注册与发现了，需要把 eureka 相关的依赖与注解删除掉。

删除 eureka client 的 @EnableDiscoveryClient 注解以及以下依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

另外 `application.yml` 配置文件也要去掉 eureka server 相关配置。

2. 由于 k8s 服务之间的调用是通过服务名称:端口的形式，所以 feign client 上需用加上 url 参数，例如：

```
@FeignClient(value = "xxxService", url = "xxxService:8080")
```

xxxService 就是后面要部署的 k8s 服务名称，8080 端口是 xxxService k8s 服务暴露的端口，可以自定义。

3. 为了我们修改 springboot 的 application.yml 文件之后，k8s 能自动感知然后重新自动部署应用，我们可以用 configmap 资源来维护我们的配置文件。需要作出以下改动：

微服务内部只保留 bootstrap.yml 文件，其他 application.yml 文件在统一的地方维护后续用于生成 configmap 资源。bootstrap.yml 文件示例：

```
spring:
  application:
     name: xxxService
  profiles:
    active: #spring.profiles.active#
  servlet:
    multipart:
      max-request-size: 100MB
      max-file-size: 100MB
  devtools:
    restart:
      enabled: true   # 设置开启热部署
  cloud:
    kubernetes:
      reload:
        enabled: true
        mode: polling
        period: 5000
      config:
        sources:
          - name: ${spring.application.name}
management:
  endpoint:
    restart:
      enabled: true
    health:
      enabled: true
    info:
      enabled: true
```

新增以下依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
    <version>1.0.1.RELEASE</version>
</dependency>
```

这个依赖的用途就是让 k8s 来管理我们的配置文件。

### 编写 Dockerfile

我们需要将 springboot 应用打包成镜像，然后上传到镜像仓库中，后面部署到 k8s 中需要指定镜像地址。Dockerfile 文件示例：

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD xxxService-1.0.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

将镜像打包到仓库这里就不展开了。

### 安装 kubectl 和 minikube

这个步骤可以参考[官方文档](https://kubernetes.io/docs/tasks/tools/install-minikube/)

### 创建 configmap 资源

假设我们将 springboot 的配置文件维护在 git 仓库中，我们就可以在安装 minikube 的机器上 git clone 下来，运行以下命令，就可以将配置文件维护在 configmap 中了。

```
kubectl create configmap xxx --from-file=application.yml
```

但是后面我们部署服务的时候，默认账户下 pod 应该是没有权限读取到 configmap 中的内容的，所以我们需要创建有权限的 serviceaccount。可以参考一下[这篇文章](https://juejin.im/post/5d71b1b4518825462823825a)

### 创建 deployment 资源

创建 deployment 资源很简单，只要指定镜像地址，运行一个简单的命令即可。

```
kubectl create deployment xxxService --image=镜像地址
```

但是这样默认创建的 deployment 资源还是有点小问题的：

1. 还没有配置读取我们上面创建的 configmap
2. 由于我用的是阿里云私有镜像仓库，没有配置用户密码会拉取镜像失败

对于第一个问题，我们需要将 configmap 挂在到容器中。参考配置：

```
spec:
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: xxx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        volumeMounts:
        - mountPath: /deployments/config
          name: easygo-device
          readOnly: true
      volumes:
      - name: demo-config
        configMap:
          name: demo-configmap
          items:
            - key: application.yml
              path: application.yml 
      imagePullSecrets:
      - name: aliyun-docker-hub

```

对于第二问题，我们需要创建 secret 资源来保存我们私有仓库的用户名密码，然后配置在 deployment 资源上，即上面的 imagePullSecrets 属性。创建 secret 资源比较简单，见下面的命令。

### 创建 secret 资源

```
kubectl create secret docker-registry 资源名称 --docker-server=仓库地址 --docker-username={UserName} --docker-password={Password} --docker-email={Email}
```

资源名称就是上面配置的 aliyun-docker-hub ，名称可自定义。

### 暴露服务给集群外部调用

默认情况下，k8s 集群外部是无法访问到集群内部的服务，所以我们需要将服务暴露出去，方式有很多种。具体可以看下[这篇文章](https://jimmysong.io/posts/accessing-kubernetes-pods-from-outside-of-the-cluster/)

最简单的方式是 NodePort，比如将 xxx 服务暴露出去，运行以下命令即可：

```
kubectl expose deployment xxx --type=NodePort --port=8080
```

然后我们就可以通过节点IP加上面暴露的端口即可访问。


而常用的方式是通过 Ingress 资源暴露，需要创建 Ingress 资源和部署 Ingress Controller。

先看一下官方的 [Ingrss 介绍](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)

然后 Ingress Controller 可以用 nginx ingress，部署方式见[官方文档](https://kubernetes.github.io/ingress-nginx/deploy/)

### 结语

本文只是介绍了 SpringCloud 迁移到 k8s 的大致流程，不是很完善，按照上面的步骤来弄应该还是会踩点坑。