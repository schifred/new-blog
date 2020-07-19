---
title: 从 nacos 看领域驱动设计
category:
  - 计算机科学
  - 软件架构
tags:
  - 计算机科学
  - 软件架构
  - nacos
  - 领域驱动设计
keywords: 'nacos,领域驱动设计,架构'
abbrlink: 2f43f2bf
date: 2020-01-23 00:00:00
updated: 2020-01-23 00:00:00
---

按 [Nacos 官网](https://nacos.io/zh-cn/index.html) 的说法，它是一个提供便捷的服务发现、管理和配置平台。推敲 Nacos 的出产，首先它基于问题域思考所需实现的功能特性和非功能特性；再由特性思忖到逻辑架构图、领域模型、部署架构图、类视图等架构层面；再结合特性和架构图深入业务场景，完善功能实现策略；然后从开发生态这个宏观视角寻味 Nacos 需要支持的语言、技术栈；最后从市场投放这个目标视角总结 Nacos 的各种优势，并予以战略上的肯定。可以推想，Nacos 基于领域模型设计，比领域模型走得更远。

![image](nacosMap.jpg)

### 一句话需求

[Nacos](https://github.com/alibaba/nacos) 充当微服务中的注册中心和配置中心。

当巨石项目被切割成多个支持动态扩展的微服务后，各个微服务的调用地址和数量都是动态可变的，注册中心的核心功能就是维护可调用的服务清单。遵循 C/S 架构，server 服务器维护着 client 可调用服务清单，并提供接口给 client 以查询其他服务信息；client 客户端一方面会将自己注册到 server 上，另一方面会从 server 上获取依赖的其他服务信息。常见的注册中心有 Eureka、Zookeeper、Consul、Dubbo。应用在不同环境中会有不同的配置，配置中心的目的即在于提供不同的配置能力。常见的配置中心有 spring cloud config、Apollo、Disconf、Diamond。

### 领域模型

#### 注册中心

注册中心基于以下概念：Service 服务、Service Provider 服务提供者、Service Consumer 服务消费者、Service Metadata 服务元数据、Service Registry 服务注册中心。由服务提供者提供服务、实例和元数据信息，并将这些内容添加到注册中心，再由服务消费者查询消费。服务下割集群，集群下挂载指定 ip、port 的实例（实例默认挂载在默认集群下，也可以创建虚拟集群管理实例）；服务上设分组。元数据包含服务端点、服务标签、服务版本号、服务实例权重、路由规则、安全策略等。Nacos 会对实例进行健康检查；当健康的实例占服务总实例比重小于指定阈值时，Nacos 将不会应用实例权重和路由规则，而是将可能不健康的实例推送给消费者。Nacos 团队在 [Nacos服务发现控制台预览 2018.10.2](https://nacos.io/en-us/blog/discovery-console.html) 这篇文章中点明了服务 - 集群 - 实例模型的界面设计。下图是包含模型结构和主要功能的领域模型。

![image](service.jpeg)

### 配置中心

配置中心基于以下概念：Configuration 配置、Configuration Management 配置管理。配置中心的主要功能在于管理应用所需的配置。单条配置（如日志级别）构成一个配置项；多个配置项通过配置集统筹；一个配置集通过配置集 id 定位；配置集上设分组，以便多个应用使用相同的配置集。对于配置管理，Nacos 还提供或规划了灰度发布、版本管理、快速回滚、监听查询、推送轨迹、权限控制、敏感配置的加密存储等功能。当配置被 client 拉取到时，Nacos 的客户端 SDK 会在本地生成配置快照，以作容灾处理；配置快照会在特定时间进行更新，但不会过期。[Nacos 帮我们解决什么问题？—— 配置管理篇] 这篇文章道明了配置中心的存在价值，上文也有简要说明。下图是包含模型结构和主要功能的领域模型。

![image](configuration.jpeg)

### 命名空间

namespace 命名空间用于区分不同租户和不同环境。命名空间下挂服务分组、配置分组。默认的命名空间为 public，分组默认是 DEFAULT_GROUP。Nacos 控制台可以添加新的命名空间，随后在 client 中即可配置命名空间的 id（id 须经解析以获得实际的命名空间名，然后从 server 中获得配置），指定服务所属哪个命名空间或服务使用哪个命名空间的配置。[Namespace, endpoint 最佳实践](https://nacos.io/zh-cn/blog/namespace-endpoint-best-practices.html) 描述着命名空间的设计背景和方案。下图是命名空间的模型结构。

![image](namespace.jpeg)

### 逻辑流程

上节所能感知的是 Nacos 控制台的界面形式（实现了什么），但不能感知 Nacos 的内部（怎么实现的）。本节结合下一节，所要表述的是 Nacos 是怎么实现。本节仅介绍注册中心的逻辑流程。

注册中心的功能分为两部分：服务注册、服务发现。下图下半部分即服务注册的流程，标名的客户端作为服务提供者将自己注册到 Nacos，Nacos 注册中心即会生成对应的一份实例配置；服务注册成功后，服务提供者与注册中心维持心跳，以保证将最新的服务信息推送到注册中心。上半部分即服务发现的流程，其一标名的客户端作为服务消费者可以从注册中心主动获取服务实例信息；其二消费者可以订阅注册中心的服务，那样会在客户端本地维护一份服务列表（通过事件机制予以更新），客户端从本地获取服务实例信息。[Nacos 服务注册与发现原理分析](https://www.jianshu.com/p/61608ff86344) 这篇文章道明了服务注册与发现的主流程。

![image](configuration.png)

[Nacos 注册中心的设计原理详解](https://www.infoq.cn/article/B*6vyMIKao9vAKIsJYpE?from=timeline&isappinstalled=0) 这篇文章道明了 Nacos 在数据隔离、数据一致性（多注册中心的前提下）、负载均衡、健康检查、集群扩展性上的设计原理。

### 整体架构

![image](jiagou.png)

在 nacos-core 核心中，与上文相关的模块有：

* 插件机制：实现服务管理、配置管理、元数据管理三个模块可分可合能力，实现扩展点 [SPI 服务提供发现机制](http://blog.itpub.net/69912579/viewspace-2656555/)。
* 事件机制：实现异步化事件通知，sdk 数据变化异步通知等逻辑。
* 回调机制：sdk 通知数据，通过统一的模式回调用户处理。接口和数据结构需要具备可扩展性。
* 推送通道：解决 server 与存储、server 间、server 与 sdk 间推送性能问题。
* 寻址模式：解决 ip、域名、nameserver、广播等多种寻址模式，需要可扩展。
* 流量管理：按照租户，分组等多个维度对请求频率，长链接个数，报文大小，请求流控进行控制。

这些核心模块撑起了上文所说的功能模块的逻辑流程。

在接口层之上、数据层之下，是 Nacos 所要支持的生态。

![image](shengtai.png)

[阿里巴巴服务注册中心产品ConfigServer 10年技术发展回顾](https://nacos.io/zh-cn/blog/alibaba-configserver.html) 这篇文章道明了 Nacos 的前世今生，各阶段致力于解决的问题。[Nacos 规划](https://nacos.io/zh-cn/docs/roadmap.html) 简述了 Nacos 今后的发展。

后记
项目使用了 Nacos，我发现在了解 Nacos 的过程中，最有意思的是探究其破土出芽的过程。虽然这一过程有点自不量力，很多东西都不能消化吸收，但是却能助我看见领域驱动设计的应用。正好最近在看《领域驱动设计 —— 软件核心复杂性的应对之道》，工作中也在接触一个探索型项目，总结一套可交流的建模语言、构建产品的一般流程很有价值似的。通过上述文字，我所能领会到的构建产品的一般流程为：

* 一句话描述需求，阐明核心功能模块。
* 构造领域模型，剖析核心流程。
* 统筹整体架构，宏观上挖掘生态、市场。
* 分解功能模块，分工、规划、细化。

我觉得这样的流程可以应用工程实践上。整理这篇文章的目的大概在于此吧。至于“细节处见真章”方面，还真是让人愧不自啊。而 Nacos 在 spring 项目中的应用，可以参考官方文档 [Nacos Spring Cloud 快速开始](https://nacos.io/zh-cn/docs/quick-start-spring-cloud.html)，这里不作说明。

### 参考

[Nacos 官网](https://nacos.io/zh-cn/)
[Nacos 服务注册与发现原理分析](https://www.jianshu.com/p/61608ff86344)
[Nacos 注册中心的设计原理详解](https://www.infoq.cn/article/B*6vyMIKao9vAKIsJYpE?from=timeline&isappinstalled=0)
[Nacos 环境隔离](https://nacos.io/zh-cn/blog/address-server.html)
[Nacos Sync 的设计原理和规划](http://blog.itpub.net/69922229/viewspace-2644195/)
[Nacos 权限控制设计方案](https://nacos.io/zh-cn/blog/access%20control%20design.html)