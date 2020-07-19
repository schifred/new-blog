---
title: spring boot（五）：数据库
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 63b17f08
date: 2019-03-17 04:00:00
updated: 2019-03-17 04:00:00
---

Spring Boot 中整合 mybaits 有两种方式：基于注解或者基于 xml 配置。

## mybatis

### 添加依赖

pom.xml 添加 mysql, mybatis 依赖。

```xml
<!-- mysql 连接依赖 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>
```

### 配置文件

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

# 基于 xml 配置添加
mybatis:
  mapper-locations: classpath:mapping/*Mapper.xml
  type-aliases-package: com.example.demo
```

### mapper

编写实体类。

```java
public class User {
    private Integer id;
    private String nickName;

    public Integer getId(){
        return id;
    }

    public void setId(Integer id){
        this.id = id;
    }

    public String getNickName(){
        return nickName;
    }

    public void setNickName(String nickName){
        this.nickName = nickName;
    }
}
```

#### 基于注解

```java
// dao/mapper 层制作 mapper 文件
@Mapper
public interface MybatisUserAnnotationMapper {
    @Select("SELECT id,nickname FROM demo.user WHERE id = #{id}")
    User getUser(@Param("id") Integer id);

    @Insert("INSERT INTO demo.user(nickname)")
    void createUser(Map<String, Object> reqMap);

    @Update("UPDATE demo.user SET nickname = #{nickName} WHERE id = #{id}")
    void updateUser(@Param("id") Integer id, @Param("nickName") String nickName);

    @Delete("DELETE FROM demo.user WHERE id = #{id}")
    void delete(@Param("id") Integer id);
}
```

#### 基于 xml 配置

```java
// dao/mapper 层制作 mapper 文件
@Mapper
public interface MybatisUserAnnotationMapper {
    User getUser(@Param("id") Integer id);

    void createUser(Map<String, Object> reqMap);

    void updateUser(@Param("id") Integer id, @Param("nickName") String nickName);

    void delete(@Param("id") Integer id);
}
```

```xml
<!-- resources/mapper 中添加 mapper.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.example.demo.dao.mapper.MybatisUserAnnotationMapper">
    <select id="getUser" resultType="com.example.demo.dao.entity.User">
        SELECT id,nickname FROM demo.user WHERE id = #{id}
    </select>

    <insert id="createUser" parameterType="java.util.Map">
        INSERT INTO demo.user(nickname) VALUES (#{nickname})
    </insert>

    <update id="updateUser">
      UPDATE demo.user SET nickname = #{nickName} WHERE id = #{id}
    </update>


    <delete id="delete">
      DELETE FROM demo.user WHERE id = #{id}
    </delete>
</mapper>
```

## druid

druid 是阿里推出的数据库监控驱动。

### 添加依赖

```xml
<!-- druid 数据库监控驱动 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.9</version>
</dependency>
```

### 配置文件

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: SELECT 1 FROM DUAL
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true
    maxPoolPreparedStatementPerConnectionSize: 20
    filters: stat,wall,log4j
    connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
    useGlobalDataSourceStat: true
```

### 添加 DruidConfig 文件

```java
@Configuration
public class DruidConfig {
    private Logger log = Logger.getLogger(LogAspect.class);

    @Bean
    @ConfigurationProperties("spring.datasource")
    public DataSource druidDataSource() {
        return new DruidDataSource();
    }

    @Bean
    public ServletRegistrationBean druidStatViewServlet() {
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(new StatViewServlet(), "/druid/*");
        registrationBean.addInitParameter("allow", "127.0.0.1");// IP白名单 (没有配置或者为空，则允许所有访问)
        registrationBean.addInitParameter("deny", "");// IP黑名单 (存在共同时，deny优先于allow)
        registrationBean.addInitParameter("loginUsername", "admin");
        registrationBean.addInitParameter("loginPassword", "123456");
        registrationBean.addInitParameter("resetEnable", "false");
        return registrationBean;
    }

    @Bean
    public FilterRegistrationBean druidWebStatViewFilter() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean(new WebStatFilter());
        registrationBean.addInitParameter("urlPatterns", "/*");
        registrationBean.addInitParameter("exclusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");
        return registrationBean;
    }
}
```

localhost:9090/druid/index.html 登录访问监控数据。

## 参考

[Homebrew介绍和使用](https://www.jianshu.com/p/de6f1d2d37bf)
[Homebrew安装Mysql步骤及注意事项（配合Navicat使用）](https://blog.csdn.net/normalizer/article/details/83478834)
[mysql 报错Authentication method ‘caching_sha2_password’ is not supported](https://blog.csdn.net/u011583336/article/details/80999043)
[springboot中使用Druid](https://www.jianshu.com/p/e3cd2e1c2b0c)