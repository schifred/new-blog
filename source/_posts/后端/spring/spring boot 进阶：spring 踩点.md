---
title: spring boot 进阶：spring 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 5f8c5aae
date: 2019-11-16 06:00:00
updated: 2019-11-16 06:00:00
---

### 前言

在我所知的前端脚手架中，dawn 使用中间件的方式串联任务流，nowa 使用子命令的方式串联任务流，umi 使用插件的方式串联任务流。三者中较为特别的是，umi 在提供打包构建、测试、mock 服务器等能力之外，它还以根据配置制作入口文件的方式，集成了路由、限定了前端代码的结构。umi 中的集成意识也许来自于 dva。无论从 dva （由状态管理器入手的）前端应用框架到 umi 脚手架，还是从 antd 组件库到 ant pro 模板工程，蚂蚁工程师们抛出的命题总是一环扣一环。我只觉得，问题会引导人去解决，开阔的应用场景会激发问题，这是一条良性的道路。如果应用场景受限，问题得不到暴露，那就要凭技术上的储备去探知下一个去脉。因此，我学习 spring 的目的在于寻找创新点，手法也是不求甚解。

### IoC

在 spring 中，控制反转就是依赖注入。当 Class B 作为 Class A 的依赖时，控制反转会将 B 实例注入到 A 实例中。其实现过程是可以推想的：首先使用 Bean 容器管理 bean 的生命周期，然后从 xml、注解或 java 配置构成的元数据中解析出 bean 的依赖关系，最后按照依赖关系将 B 实例组装到 A 实例中。在 spring 中，作为 Bean 容器的是 BeanFactory；ApplicationContext 作为 BeanFactory 的超类，额外提供对 AOP、资源加载、国际化、事件发布（通过 ApplicationContext 中转提供 bean 之间消息通信的能力）、特定应用层上下文的支持。在这套机制的基础上，spring 既对 bean 注入的多种情形作了适配（也就是以超强的模式解析能力获取元数据），又可以通过 ApplicationContext 实例访问 bean。由此 spring 提供了声明式编程的能力。

![image](container-magic.png)

在上述过程中，spring 需要解决的问题包含：通过 xml、注解或 java 配置 bean 的依赖关系；通过构造函数、静态工厂函数、setter 方法注入依赖；注入时须对依赖 bean 作类型转换和顺序处理，并管理 bean 的实例化、销毁等生命周期；解决 bean 的循环依赖问题（通过预实例化或使用 setter 注入）；实现 bean 的自动装配；扫描文件夹以获取 bean；支持元数据的各种便捷、组合、继承、复杂配置（抽象为 BeanDefinition）；以 @Scope 限定 bean 的作用域（singleton 单例、prototype 调用时创建、request 针对请求创建 bean、session 针对 session 创建 bean、globalSession、自定义作用域等）；以 @Profile 区分不同环境。spring 允许 bean 实现 BeanNameAware、BeanFactoryAware、ApplicationContextAware 等接口，回调或访问 IoC 容器的能力。

![image](bean.png)

在 bean 实例化、初始化、依赖解析等过程中，BeanPostProcessor、BeanFactoryPostProcessor、FactoryBean 实现类会以钩子形态被执行。BeanPostProcessor 用于在 bean 实例化后定制 bean；BeanFactoryPostProcessor 用于定制 bean 的配置元数据；FactoryBean 用于定制 bean 的实例化逻辑。以 BeanPostProcessor 为例，其核心处理流程为：在 IoC 容器实例化 bean 后，再由 BeanPostProcessor 处理这些 bean，比如进行校验或者封装（包含 AOP 逻辑）。

![image](BeanPostProcessor.png)

### Resource

Resource 接口抽象了对资源访问的操作，其实现类包含 UrlResource、ClassPathResource、FileSystemResource、ServletContextResource、InputStreamResource、ByteArrayResource 等。ApplicationContext 接口的实现类同时实现了 Resource 接口，加载资源时通过前缀使用不同的 Resource 实现类。

### Bean

spring 为 Bean 提供了以下组件能力：Validator 校验器（Validator 本身与 bean 解耦，需要通过 DataBinder 绑定；校验错误由 spring 机制国际化）、BeanWrapper（用于设置或获取 bean 的属性，支持属性变更时的事件绑定）、PropertyEditor（用于作类型转换）、Converter（用于作类型转换）、Formatter（用于作类型转换，支持国际化）、DataBinder（用于为 bean 绑定 Validator 等）。

### SpEL

Spring 表达式语言（简称 SpEL）由 spring 解析字符串表达式，可用于查询和操作数据，其应用点在于 @Value 值设置、解析表达式等。

### AOP

AOP 框架为 spring 提供了强力的中间件解决方案。spring 切面基于代理实现，因此可以在目标对象方法执行前后执行额外的拦截器方法（或建议 advice）。如下图所示，先执行代理方法，期间调用目标对象的实际方法。

![image](aop-proxy-call.png)

### 后记
本文聚焦于解读一个成熟框架在设计上包含了哪些功能，以期拓展思维，因此所述内容不免点到为止。从延伸面上，依赖注入见于 angular、redux-react；模式解析见于 schema form/page；bean 校验和转换可应用于前端数据模型；表达式语言见于模板引擎；AOP 可应用于设计中间件、拦截器。

### 参考
[The IoC Container](https://docs.spring.io/spring/docs/5.2.2.BUILD-SNAPSHOT/spring-framework-reference/core.html)
[IoC容器和Bean](https://www.jianshu.com/p/c3204049cf02)
[Spring Resource框架体系介绍](https://blog.csdn.net/javazhiyin/article/details/93558802)
[属性编辑器PropertyEditor](https://blog.csdn.net/shenchaohao12321/article/details/80295371)