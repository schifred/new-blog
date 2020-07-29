---
title: spring boot 进阶：spring boot 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 1cc17133
date: 2020-01-31 06:00:00
updated: 2020-01-31 06:00:00
---

spring cloud 是一个基于 spring boot 的服务治理框架，它由众多服务治理组件构成：

* 注册中心：Erueka、Zookeeper、Consul 等用于注册、发现服务。
* 配置中心：Spring Cloud Config 提供分布式系统的配置管理功能（运行时更新配置文件需要 refresh 才能重新加载配置）。
* 网关（外部调用）：Zuul、Spring Cloud Gateway 为动态实例提供外部调用入口，可以基于横切关注点实现权限校验、监控指标、负载均衡等功能。
* 内部调用：OpenFeign 声明式 RESTful 网络请求客户端。
* 断路器：Hystrix 隔离调用 N 次失败的不可用服务，避免服务级联雪崩。Hystrix-dashboard 查看各 Hystrix Command 的请求响应时间，请求成功率等数据，只能查看单个应用的服务信息。Turbine 能查看系统内多个服务的调用数据。
* 负载均衡：Ribbon。
* 分布式消息：Spring Cloud Stream、Spring Cloud Bus。Spring Cloud Bus 可用于促使客户端重新拉取配置，即 Spring Cloud Config 相关配置文件提交到代码库时，webhook 通知 Spring Cloud Bus；由 Spring Cloud Bus 促使订阅消息的客户端重新从 ConfigServer 拉取配置。
* 安全控件：Spring Cloud Security 基于 OAuth2.0 的安全控件。
* 链路监控：Zipkin、Dapper、Eagleeye、Spring Cloud Sleuth 通过数据埋点监控微服务性能及调用链路。

![image](spring_cloud.jpg)

### spring cloud 公共库

#### spring cloud 上下文

spring cloud 有两个上下文：application.yml 应用上下文、bootstrap.yml 引导上下文。依据 spring 层级上下文机制，引导上下文作为应用上下文的父级，具有较低的优先级。

### 参考

[基于 Spring Cloud 的分布式架构体系](https://www.cnblogs.com/powerwu/articles/9776357.html)
[Spring Cloud 微服务架构学习笔记与示例](https://www.cnblogs.com/edisonchou/p/java_spring_cloud_foundation_sample_list.html)
[深入理解 Spring Cloud 引导上下文](http://www.mamicode.com/info-detail-2274944.html)