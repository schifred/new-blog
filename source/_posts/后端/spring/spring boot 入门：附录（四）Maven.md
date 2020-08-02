---
title: spring boot 入门：附录（五） Maven
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 692189cc
date: 2019-03-17 17:00:00
updated: 2019-03-17 17:00:00
---

### pom.xml

pom.xml 如同 package.json，groupId、artifactId、version 用于锁定模块。按[阿里巴巴Java开发手册](https://github.com/alibaba/p3c/blob/master/p3c-gitbook/%E5%B7%A5%E7%A8%8B%E7%BB%93%E6%9E%84/%E4%BA%8C%E6%96%B9%E5%BA%93%E4%BE%9D%E8%B5%96.md)中约定，GAV 遵从以下格式：

* GroupID格式：com.{公司/BU }.业务线.[子业务线]，最多4级。
* ArtifactID格式：产品线名-模块名。语义不重复不遗漏，先到中央仓库去查证一下。
* Version格式：以二方库版本号为例，其命名方式为，主版本号.次版本号.修订号。主版本号意为产品方向改变，向下不兼容；次版本号保持相对兼容性，增加主要功能特性；修订号保持完全兼容性。起始版本号必须为：1.0.0。子模块的版本号须与父模块保持一致。

最常见的是以下标签：

* packaging：设置打包机制，默认为 jar，父模块使用 pom
* properties：全局可用的属性信息，一般用于管理依赖的版本号
* dependencies、dependency：声明依赖。exclusions 剔除包的依赖；scope 依赖的作用范围；provided 引入，如同 peerDependency
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
* mvn dependency:tree 查看项目的依赖关系
* mvn dependency:resolve-plugins 根据 pom.xml 下载或更新依赖
