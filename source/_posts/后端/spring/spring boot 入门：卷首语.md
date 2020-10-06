---
title: spring boot 入门：卷首语
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 9082f32
date: 2019-03-17 00:00:00
updated: 2019-03-17 00:00:00
---

本系列文章仅就 spring boot 作指南性介绍，未作深入。

* [搭建项目](/archives/b16f5632/)
* [application.yml 配置文件](/archives/6fd0dc6f/)
* [logback 日志](/archives/89954b97/)
* [controller 控制器](/archives/7981e696/)
* [参数校验与错误捕获](/archives/c158ade3/)
* [使用 thymeleaf 渲染页面](/archives/817b8053/)
* [filter 过滤器](/archives/68ab87cc/)
* [aop 切面编程](/archives/ad1e9018/)
* [mybatis 访问数据库](/archives/63b17f08/)
* [定时任务](/archives/2290be3f/)
* [使用 spring cache](/archives/96b22f49/)
* [使用 redis 缓存](/archives/e219e00a/)
* [使用 RestTemplate 发送 http 请求](/archives/7d49460/)
* [接入 swagger2](/archives/d9e408c0/)
* [测试](/archives/b600fbee/)
* [常见功能示例](/archives/363acb26/)
* [常见问题](/archives/9999f3ca/)
* 附录
  * [Maven](/archives/692189cc/)
  * [IDEA](/archives/b36cf34f/)
  * [mysql](/archives/73386e0/)

demo 戳 [这里](https://github.com/Alfred-sg/spring-boot-demo)，工程目录结构如下：

```md
.
│  pom.xml # Maven 工程模块、依赖、构建管理
│  
└─src
    ├─main
    │  ├─java
    │  │  └─com
    │  │      └─example
    │  │          └─demo
    │  │              │  DemoApplication.java # 启动类
    │  │              │      
    │  │              ├─config # @Configuration 标注的配置类，可用于生产 bean
    │  │              ├─dal
    │  │              │  └─entity # mybatis 实体类
    │  │              │  └─mapper # mybatis *Mapper.java 接口
    │  │              │      
    │  │              ├─service # @Service 标注的业务逻辑层
    │  │              │      
    │  │              ├─dto # service、controller 层向外传输实体
    │  │              │      
    │  │              └─controller # @Controller 标注的控制器层
    │  │                      
    │  └─resources
    │      │  application.yml # 配置文件
    │      │  logback-config.xml # logback 日志配置文件
    │      │  
    │      ├─mappers # mybatis *Mapper.xml 文件
    │      └─templates # html 模板文件存放路径
    |
    └─test # 测试类存放路径
```

[阿里巴巴Java开发手册](https://github.com/alibaba/p3c/blob/master/p3c-gitbook/%E5%B7%A5%E7%A8%8B%E7%BB%93%E6%9E%84/%E5%BA%94%E7%94%A8%E5%88%86%E5%B1%82.md)中约定的工程结构为：

![image](alibabaLevel.png)

* 开放接口层：可直接封装Service方法暴露成RPC接口；通过Web封装成http接口；进行网关安全控制、流量控制等。
* 终端显示层：各个端的模板渲染并执行显示的层。当前主要是velocity渲染，JS渲染，JSP渲染，移动端展示等。
* Web层：主要是对访问控制进行转发，各类基本参数校验，或者不复用的业务简单处理等。
* Service层：相对具体的业务逻辑服务层。
* Manager层：通用业务处理层，它有如下特征：
1） 对第三方平台封装的层，预处理返回结果及转化异常信息；
2） 对Service层通用能力的下沉，如缓存方案、中间件通用处理；
3） 与DAO层交互，对多个DAO的组合复用。
* DAO层：数据访问层，与底层MySQL、Oracle、Hbase等进行数据交互。
* 外部接口或第三方平台：包括其它部门RPC开放接口，基础平台，其它公司的HTTP接口。

以及分层领域模型：

* DO（Data Object）：与数据库表结构一一对应，通过DAO层向上传输数据源对象。
* DTO（Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象。
* BO（Business Object）：业务对象。由Service层输出的封装业务逻辑的对象。
* AO（Application Object）：应用对象。在Web层与Service层之间抽象的复用对象模型，极为贴近展示层，复用度不高。
* VO（View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象。
* Query：数据查询对象，各层接收上层的查询请求。注意超过2个参数的查询封装，禁止使用Map类来传输。

阿里巴巴开发手册另外提供了 IDEA 插件，可以戳 [这里](https://github.com/alibaba/p3c/blob/master/idea-plugin/README_cn.md)。