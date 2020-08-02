---
title: spring boot 入门：附录（一） Maven
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: b600fbee
date: 2019-03-17 12:00:00
updated: 2019-03-17 12:00:00
---

### pom.xml

pom.xml 如同 package.json，groupId、artifactId、version 用于锁定模块。最常见的是以下标签：

* packaging：设置打包机制，默认为 jar，父模块使用 pom
* properties：全局可用的属性信息，一般用于管理依赖的版本号
* dependencies、dependency：声明依赖。exclusions 剔除包的依赖；scope 依赖的作用范围
* parent：父模块
* modules：包含的子模块
* build：构建配置，可指定 jdk 版本号

更多内容可戳 [pom.xml 详解](https://zhuanlan.zhihu.com/p/100106971)。

#### 常见问题

Q：Element 'dependencies' cannot have character [children], because the type's content type is element-only.报错
A：原因是有奇怪的字符，须查验 pom.xml 文件

### Maven

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
