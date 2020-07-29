---
title: spring boot 进阶：mybatis-generator
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 338350a2
date: 2020-04-21 06:00:00
updated: 2020-04-21 06:00:00
---

### 步骤

1. 创建 maven 项目，如 generator-mapper。
2. 配置 pom.xml。
3. 配置 resources/generatorConfig.xml。
4. maven 面板点击 Plugins - mybatis-generator:genrate。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>generate_mapper</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>generate_mapper</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <plugin>
                <groupId>org.mybatis.generator</groupId>
                <artifactId>mybatis-generator-maven-plugin</artifactId>
                <version>1.3.5</version>
                <configuration>
                    <verbose>true</verbose>
                    <overwrite>true</overwrite>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.mybatis.generator</groupId>
                        <artifactId>mybatis-generator-core</artifactId>
                        <version>1.3.2</version>
                    </dependency>

                    <dependency>
                        <groupId>mysql</groupId>
                        <artifactId>mysql-connector-java</artifactId>
                        <version>8.0.15</version>
                    </dependency>
                </dependencies>
            </plugin>
        </plugins>
    </build>
</project>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="MysqlTables" targetRuntime="MyBatis3">

        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true"/>
            <!-- 将数据库中表的字段描述信息添加到注释 -->
            <property name="addRemarkComments" value="true"/>
            <!-- 是否生成注释代时间戳-->
            <property name="suppressDate" value="false" />
        </commentGenerator>

        <!-- jdbc链接信息 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&amp;characterEncoding=UTF8"
                        userId="root" password="123456">
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- 生成PO类的位置 -->
        <javaModelGenerator targetPackage="com.example.generate_mapper.dal.domain"
                            targetProject="src/main/java">
            <property name="enableSubPackages" value="false"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- mapper映射文件生成的位置 -->
        <sqlMapGenerator targetPackage="resources.mapper"
                         targetProject="src/main">
            <property name="enableSubPackages" value="false"/>
        </sqlMapGenerator>

        <!-- mapper接口生成的位置 -->
        <javaClientGenerator type="XMLMAPPER"
                             targetPackage="com.example.generate_mapper.dal.mapper"
                             targetProject="src/main/java">
            <property name="enableSubPackages" value="false"/>
        </javaClientGenerator>

        <!-- 指定要生成的表 -->
        <table tableName="test" domainObjectName="Test"
               enableCountByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false"
               enableUpdateByExample="false"></table>
    </context>
</generatorConfiguration>
```

### QA

* Communications link failure 报错：数据库端口配置错误等，参看 [“Communications link failure”错误](https://blog.csdn.net/qq_27471405/article/details/80921846)
* Unknown system variable ‘query_cache_size’ 报错：mysql 驱动程序版本与 mysql 服务器版本不一致，查看 [Unknown system variable ‘query_cache_size’](https://blog.csdn.net/qq_21870555/article/details/80711187)