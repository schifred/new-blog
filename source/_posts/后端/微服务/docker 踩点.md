---
title: docker 踩点
category:
  - 微服务
  - 部署
tags:
  - 微服务
  - 部署
keywords: 'docker,部署'
abbrlink: 23e9efd5
date: 2019-02-16 00:00:00
updated: 2019-02-16 00:00:00
---

虚拟机和容器都是为了将实体机拆分成多块，以便分块运行不同的应用。

![image](d1.png)

虚拟机技术（xem、kvm、vmware、virtualbox）需要模拟整台机器，包括硬件、操作系统等。虚拟机一旦开启，预分配给它的资源将全部被占用。虚拟机上再运行应用，并安装必要的二进制包和库。

容器技术能和宿主机共享硬件资源及操作系统，能实现资源的动态分配。容器内部安装依赖和应用，在宿主机中以分离的进程运行。Docker 基于 Linux 容器封装。Linux 容器没有模拟一个操作系统，而是对进程套了一层保护层，它所使用的资源都是虚拟的，这样就能与底层系统进行隔离。Docker 支持将软件编译成镜像（image）（镜像是一个包含一堆只读层的文件系统），并在镜像中做好对软件的配置，镜像经发布后，使用者就可以下载并运行这个镜像（运行中的镜像也被称为容器 Container）（容器也是多层机构的文件系统，只是最上面那层支持读写操作）。通过镜像可以实现交付环境的一致性；Dockerfile 用于记录容器构建过程，可以在集群中实现快速分发和部署。

各大云计算平台（PaaS 平台即服务，提供存储、数据库、网络、负载均衡、自动扩展等功能）均支持 Docker 容器技术，比如阿里云、百度云、Cloud Foundry、HeroKu、DigitalOcean、OpenShift、Apache Stratos、Apache MesOS、Deis、Windows Server、Azure 等。

### Docker 命令

Docker 使用 C/S 结构（客户端/服务器）打造。客户端和服务器可以运行在同一个 Host 上，客户端也可以通过 socket 或 REST API 与远程的服务器通信。典型的客户端就是命令行工具。服务端接受到请求后，会使用路由分发工具请求交给具体的处理器，如 graphdriver 用来执行容器镜像的本地化操作；networkdriver 用来执行容器网络环境的配置；execdriver 用来执行容器内部运行的执行工作；或者从 Docker Registry 上获取镜像。

![image](d2.png)

### Dockerfile

Dockerfile 配置文件用于构建 Docker 镜像，包含四大内容：

* FROM：基础镜像（父镜像）信息指令
* MAINTAINER：维护者信息指令
* RUN、EVN、ADD、WORKDIR 等：镜像操作指令
* CMD、ENTRYPOINT、USER 等：容器启动指令

典型的 Dockerfile 文件如下（编写完 Dockerfile 后，可以通过 docker build 编译创建镜像）（jar 包也可以在应用中添加 docker-maven-plugin 插件，并指定 DOCKER_HOST 环境变量为服务器地址，再使用 mvn clean package docker:build -DskipTests 构建）：

```bash
FROM java:8 # 拉取基础镜像
VOLUME /tmp # 创建一个可以从本地主机或其他容器挂载的挂载点，一般用来存放数据库和需要保持的数据等
RUN mkdir /app # 执行命令
ADD config-1.0.0-SNAPSHOT.jar /app/app.jar # 赋值
ADD runboot.sh /app/ # 赋值
RUN bash -c 'touch /app/app.jar' # 执行命令，另一种格式是 RUN ["bash", "-c", "touch /app/app.jar"]
WORKDIR /app # 设置当前工作路径
RUN chmod a+x runboot.sh 
EXPOSE 8888 # 指定对外开放的端口
CMD /app/runboot.sh # 启动容器时的默认行为，一个 Dockerfile 里只能有一个 CMD 指令

# /app/runboot.sh
sleep 10
java -Djava.security.egd=file:/dev/./urandim -jar /app/app.jar
```

### Docker Compose

Docker Compose 可用来定义和运行多容器应用，即使用 docker-compose.yml 定义多容器，然后执行 docker-compose up -d 启动多应用。

### 参考

[这可能是最为详细的Docker入门吐血总结](https://blog.csdn.net/deng624796905/article/details/86493330)
[终于有人把 Docker 讲清楚了，万字详解！](https://zhuanlan.zhihu.com/p/89587030)
[Docker教程：Docker入门实践（精讲版）](http://c.biancheng.net/docker/)
[Docker中文文档](http://www.dockerinfo.net/document)