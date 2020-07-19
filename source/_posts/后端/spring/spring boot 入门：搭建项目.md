---
title: spring boot 入门：搭建项目
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: 'spring boot'
abbrlink: b16f5632
date: 2019-03-17 00:00:00
updated: 2019-03-17 00:00:00
---

### 创建并启动项目

创建 spring boot 项目需要先安装 [Java](https://www.oracle.com/java/technologies/javase-downloads.html), [Maven](http://maven.apache.org/download.cgi) 并配置环境变量。

spring boot 项目可以通过 [Spring Initializr](https://start.spring.io/) 创建，也可以通过 [IntelliJ IDEA](https://www.jetbrains.com/idea/download/#section=mac) 创建。使用 IDEA 创建时，依次选择 

1. create project 创建项目。
2. spring initializr 创建 spring 项目。
3. 设置 name, groupId, ArtifactId 等项目信息。groupId 是项目所属的组织，ArtifactId 是项目在组织中的唯一标识。

通过 [Spring Initializr](https://start.spring.io/) 创建，仅需要在 IDEA 中选择 open project 即可。

创建完的项目会包含启动类如下：

```java
// src/main/java 目录下的启动类，右键选择 Run/Debug 启动
@SpringBootApplication
public class DemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

对于刚创建的项目，可以在 IDEA 中点击 Maven 面板的 Reimport All Maven Projects 按钮，这样可以加载项目所需的依赖和插件。Maven 工程的依赖在 pom.xml 文件中约定，pom.xml 如同前端项目中的 package.json。上文中的 groupId、ArtifactId 一同锁定 Maven 项目的唯一性，如同 package.json 中的 name。

安装完依赖后，就可以启动项目了。在 IDEA 中，项目的启动类右键点选 Run/Debug，这样会自动生成本地调试模式时的 Configuration 配置文件并启动项目。启动成功后，根据应用配置访问页面即可，一般是 localhost:8080。

### 编写一个简单的 controller

按 [Java项目目录结构推荐](https://www.jianshu.com/p/b5ebe9fc6c3d) 的目录结构，我们可以在 controller 目录下创建如下类：

```java
// controller 目录 HelloController.java
@RestController
public class HelloController {
    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String hello(){
        return "Hello World!";
    }
}
```

之后重新启动项目，访问 localhost:8080/hello 就可以看见 "Hello World!" 了。

### 其他

#### 多模块项目

在 IDEA 中搭建多模块项目，可以在已创建的项目右键选择 New Module 创建，它会把已有的模块作为父模块创建新的项目。子模块的 pom.xml 会继承使用父模块的 pom.xml，方便管理。同时，父模块的 pom.xml 中 modules 会包含子模块名。

更多内容可戳 [使用 idea 创建 springboot 多模块项目](https://blog.csdn.net/lhw_csd/article/details/82183008)。

#### pom.xml

pom.xml 如同 package.json，groupId、artifactId、version 用于锁定模块。最常见的是以下标签：

* properties：全局可用的属性信息
* dependency：生命依赖
* exclusions：剔除包的依赖
* parent：父模块
* modules：包含的子模块

#### Maven

上文已说明，Maven 如同前端工程中的 npm。与 npm 一样，直接从 Maven 官方下载仓库是比较慢的，这时可以通过改写 Maven 的 settings.xml 配置文件，将仓库指定为阿里云 Maven 仓库。改写配置文件的详细方式可以戳 [这里](https://www.cnblogs.com/sword-successful/p/6408281.html)。下载依赖时，首先会查找本地资源，然后查找私服镜像，最后 Maven 官方仓库。

同样的，npm 官网可以查询 npm 包详情，我们也可以在 [Maven 官网](https://mvnrepository.com/)或 [阿里 Maven](https://maven.aliyun.com/mvn/search) 查看依赖的详细信息。

在 IDEA 工具面板中，Maven 可执行以下操作（也可以通过命令行界面操作）：

* mvn compile –src/main/java 编译生成 class（target 目录下）
* mvn test –src/test/java 编译
* mvn clean 删除 target 目录
* mvn package　生成压缩文件，java 项目 jar 包，web 项目 war 包（target 目录下）
* mvn install　将压缩文件（jar 或 war 包）上传到本地仓库
* mvn deploy 将压缩文件上传私服
* mvn idea:idea 将 maven java 或 web 项目转成 idea 工程
* mvn idea:clean 清除 idea 配置，将 idea 工程转成 maven 项目

#### 使用 jetty

spring boot 默认使用 tomcat。如果想禁用，使用其他的 servlet 容器如 jetty，可修改 pom.xml，如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

#### starter pom 说明

spring boot 以约定大于配置的方式简化了 spring 工程的配置，其实现手段就是提供了大量的 starter pom。使用 IDEA 或 [Spring Initializr](https://start.spring.io/) 创建的项目，均会在 pom.xml 指定 spring-boot-starter-web 依赖。

作为 starter pom，spring-boot-starter-web 实际整合了 spring-boot-starter, spring-boot-starter-tomcat, spring-web, spring-webmvc, hibernate-validator 等包（参考 [Spring Boot Starter Web](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web/2.1.3.RELEASE)）。因为内嵌 tomcat，Spring Boot 项目就无须以 war 包形式部署。

在项目中执行 mvn dependency:tree 命令，既可以下载依赖，又可以查看依赖的结构。

更多内容可戳 [starter pom详解](https://blog.csdn.net/newbie_907486852/article/details/79798700)。

#### @SpringBootApplication 说明

@SpringBootApplication 实际是个组合注解，包含 @Configuration, @EnableAutoConfiguration, @ComponentScan。

@EnableAutoConfiguration 作用是在 classpath 中搜索所有 META-INF/spring.factories 配置文件，然后将其中  EnableAutoConfiguration 置为 true 的配置项加载到 spring 容器中，以实现包的自动配置。

@ComponentScan 使 Spring Boot 自动扫描启动类同级及以下目录，自动加载注解了 @Controller、@Service 等的 Bean 实例。

@Configuration 用于定义配置类，可以被 AnnotationConfigApplicationContext 或 AnnotationConfigWebApplicationContext 类扫描到，用于构建配置类内部的 Bean，以初始化 Spring 容器。

更多内容可戳 [@EnableAutoConfiguration 注解原理](https://www.cnblogs.com/whx7762/p/7832985.html)、 [@Configuration 详解](https://blog.csdn.net/koflance/article/details/59304090)。

#### IDEA 快捷键

* 双击 shift 当前项目的所有文件夹中查找内容
* ctrl + f 当前文件查找特定内容
* ctrl + shift + f 查找特定内容
* ctrl + n 查找类
* ctrl + shift + n 查找文件
* ctrl + r 当前文件替换内容
* ctrl + shift + r 当前项目替换内容
* shift + F6 重命名类等
* alt + insert 类中使用自动生成方法，项目中使用创建类
* alt + enter 消除黄色警告，自动优化代码
* ctrl + alt + l 代码格式化

更多内容可戳 [IDEA 快捷键](https://blog.csdn.net/xkzju2010/article/details/69487723)。

#### 常见问题

1. 项目目录结构折叠问题
    点击 Project 一行上齿轮图标，取消 Hide Empty Middle Packages 选中态

2. 使用 IDEA 启动项目时，报 Java 1.5 版本过等错误
    检查 Project Structure - Project | Modules 指定的当前项目 jdk 版本、以及 Setting - Java Compiler 指定的 Java 版本是否正确。

3. 启动时报端口号被占用
    查看并杀死占用端口号的进程 
    netstat -ano |findstr 18080
    taskkill /f /t /im 23384

4. 指定 profile 方式启动
    VM OPTIONS 添加 -Dspring.profiles.active=local
