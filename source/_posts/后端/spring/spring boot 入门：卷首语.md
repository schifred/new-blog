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

