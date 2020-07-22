---
title: spring boot 入门：配置文件
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 6fd0dc6f
date: 2019-03-17 01:00:00
updated: 2019-03-17 01:00:00
---

Spring Boot 支持使用 application.properties 或 application.yml 作为项目的配置文件，这些配置文件通常会在 src/main/resources 目录下。Spring Boot 的启动类会自动搜索并加载配置文件，以获得配置内容。搜索范围实际上包含当前目录、classpath 中的 config 包、classpath 等，且可以通过 spring.config.location 添加额外的搜索路径。

配置内容不只支持开发者编程需要，也提供给依赖使用。同时，Spring Boot 也支持分环境配置不同的内容。

### 获取配置

配置内容可以通过 @Value 注入到 Bean 中，如 @Value(“${alfred.test}”)，也可以通过 @ConfigurationProperties 注解构建 POJO 读取，如 @ConfigurationProperties(prefix = “alfred”)。

```yml
server:
  port: 9090
  
alfred:
  test: '测试'
  name: 'name'
```

对于以下配置内容，我们可以通过如下代码获取：

```java
@RestController
@RequestMapping(value = "/alfred")
public class AlfredController {
    private static String test;

    @Value("${alfred.test}")
    public void setTest(String test) {
        AlfredController.test = test;
    }

    @Value("${alfred.name}")
    private static String name;

    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String getTest(){
        return this.test;
    }
}
```

### 区分环境

针对不同的环境，Spring 支持创建 application-dev.yml, application-test.yml, application-prod.yml 文件，并在 application.yml 指定当前所采用的配置文件，这被称为 Profile 配置。

```yml
// application.yml
spring:
  profiles:
    active: dev
```

```java
private String getEnv(){
    String env = System.getProperty("spring.profiles.active");
    return env;
}
```

### 常见配置

#### 基础

```yml
server:
  port: 18080
spring:
  application:
    name: demo
  servlet:
    multipart: # 文件上传配置
      max-file-size: 100MB
      max-request-size: 100MB
```

#### druid 监控

[druid](https://github.com/alibaba/druid) 作为数据库连接池，提供了监控的能力。使用前须添加 druid-spring-boot-starter 依赖。

更多内容可戳 [Spring Boot 项目集成阿里 Druid 连接池](https://www.jianshu.com/p/ce8df857c951)、[Druid 监控功能的深入使用与配置](https://blog.csdn.net/justlpf/article/details/80728529)。

```yml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai # useSSL=true 解决JDBC版本与MySQL版本不兼容问题
    username: odic
    password: Gw32_iuX
    druid:
      # 连接池配置(通常来说，只需要修改initialSize、minIdle、maxActive
      # 如果用Oracle，则把poolPreparedStatements配置为true，mysql可以配置为false。分库分表较多的数据库，建议配置为false。removeabandoned不建议在生产环境中打开如果用SQL Server，建议追加配置)
      initial-size: 5 # 初始化物理连接个数
      min-idle: 2 # 连接池中最小空闲连接数
      max-active: 20 # 连接池中最大的活跃连接数
      max-wait: 60000 # 配置获取连接等待超时的时间
      pool-prepared-statements: false # 开启缓存preparedStatement(PSCache)
      max-pool-prepared-statement-per-connection-size: -1 # 启用PSCache后，指定每个连接上PSCache的大小
      test-on-borrow: false # 申请连接时不检测连接是否有效
      test-on-return: false # 归还连接时不检测连接是否有效
      test-while-idle: true # 如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效
      validation-query: SELECT 'x'
      time-between-eviction-runs-millis: 60000 # 检测连接的间隔时间，若连接空闲时间 >= minEvictableIdleTimeMillis，则关闭物理连接
      min-evictable-idle-time-millis: 300000 # 连接保持空闲而不被驱逐的最小时间
      filter: config,stat # 配置监控统计拦截的filters（不配置则监控界面sql无法统计），监控统计filter:stat，日志filter:log4j，防御sql注入filter:wall
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000;config.decrypt=false; # 通过connectProperties属性来打开mergeSql功能；慢SQL记录
      web-stat-filter: 
        enabled: true # 是否启用StatFilter默认值true
        url-pattern: /*
        exclusions: /*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
        session-stat-enable: false
        session-stat-max-count: 1000
        principal-session-name: admin # 从session中获取身份标识
        principal-cookie-name: admin
        profile-enable: true
      # StatViewServlet配置
      #展示Druid的统计信息,StatViewServlet的用途包括：
      #   1.提供监控信息展示的html页面
      #   2.提供监控信息的JSON API
      stat-view-servlet:
        enabled: true # 是否启用StatViewServlet默认值true
        url-pattern: /druid/* # 根据配置中的url-pattern来访问内置监控页面
        reset-enable: false # 禁止手动重置监控数据
        login-username: admin # 设置监控页面的登录名和密码
        login-password: admin
        #StatViewSerlvet展示出来的监控信息比较敏感，是系统运行的内部情况，如果你需要做访问控制，可以配置allow和deny这两个参数
        #deny优先于allow，如果在deny列表中，就算在allow列表中，也会被拒绝。如果allow没有配置或者为空，则允许所有访问
        #配置的格式：<IP> 或者 <IP>/<SUB_NET_MASK_size>其中128.242.127.1/24
        #24表示，前面24位是子网掩码，比对的时候，前面24位相同就匹配,不支持IPV6。
        #deny:xxx
        #allow: 127.0.0.1
      aop-patterns: com.example.demo.*.*.* # 对内部各接口调用的监控
```

#### mybatis

mybatis 作为持久层访问框架，简化了数据库操作。使用前须添加 mybatis-spring-boot-starter 依赖。

```yml
mybatis:
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: com.example.demo.dao
  configuration:
    map-underscore-to-camel-case: true
```

#### mybatis-pagehelper

```yml
pagehelper:
  helperDialect: mysql # 设置数据库方言
  reasonable: true # 分页合理化，即超过总条目时设为总条目数
  supportMethodsArguments: true # 支持通过 Mapper 接口参数来传递分页参数
  params: count=countSql # 设置参数映射
```

#### redis

redis 一般用于提供分布式数据缓存服务，以使应用无状态。

更多内容可戳 [SpringBoot + Redis：基本配置及使用](https://www.cnblogs.com/caizhaokai/p/11037610.html)、[redis 深度历险](https://www.jianshu.com/p/b422b153f650)、[Spring Boot 整合 Redis](https://www.cnblogs.com/yliucnblogs/p/10240576.html)。

```yml
spring:
  redis:
    database: 10 # Redis数据库索引（默认为0）
    host: 127.0.0.1 # Redis服务器地址
    port: 6379 # Redis服务器连接端口
    password: 123456 # Redis服务器连接密码（默认为空）
    ssl: false
    lettuce:
      pool:
        max-active: 50 # 连接池最大连接数（使用负值表示没有限制）
        max-wait-millis: 10000 # 连接池最大阻塞等待时间（使用负值表示没有限制）
        timeout: 0 # 连接超时时间（毫秒）
        max-idle: 10 # 连接池中的最大空闲连接
        min-idle: 5 # 连接池中的最小空闲连接
        test-on-borrow: true # 调用者获取链接时，是否检测当前链接有效性
        test-on-return: true # 向链接池中归还链接时，是否检测链接有效性
        test-while-idle: true # 调用者获取链接时，是否检测空闲超时, 如果超时，则会被移除
        time-between-eviction-runs-millis: 30000 # 空闲链接检测线程检测周期（使用负值表示不运行检测线程）
        num-tests-per-eviction-run: 10 # 空闲链接检测线程一次运行检测多少条链接
        min-evictable-idle-time-millis: 60000 # 连接在池中最小的生存时间
```

#### log

使用日志服务可先在 pom.xml 中添加 log4j-over-slf4j、jcl-over-slf4、jul-to-slf4j、logback-core 依赖。

log4j-over-slf4j 通过代理将系统中所有 log4j 日志（含第三方库的）路由到 slf4j上。jcl-over-slf4 将 apache commons logging 路由到 slf4j上。jul-over-slf4 将 java.util.logging 路由到 slf4j 上。

更多内容可戳 [log4j-over-slf4j 工作原理详解](https://blog.csdn.net/john1337/article/details/76152906)、[logback 详解](https://blog.csdn.net/Sadlay/article/details/88732271)、[logback中文手册](http://www.logback.cn/)。

```yml
logging:
  config: classpath:logback-config.xml
  file:
    path: /data
  level:
    all: debug
    root: debug
```

除了 application.yml 外，还需要配置 logback-config.xml。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- https://github.com/spring-projects/spring-boot/blob/v1.4.2.RELEASE/spring-boot/src/main/resources/org/springframework/boot/logging/logback/defaults.xml -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- 读取 application.yml 配置 -->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    <springProperty scope="context" name="LOG_BASE_PATH" source="logging.file.path"/>
    <springProperty scope="context" name="BASE_LEVEL" source="logging.level.all"/>
    <springProperty scope="context" name="ROOT_LEVEL" source="logging.level.root"/>

    <property name="LEVEL" value="${BASE_LEVEL}"/>
    <property name="LOG_PATH" value="${LOG_BASE_PATH}/logs/${APP_NAME}"/>
    <property name="ARCHIVE_LOG_PATH" value="${LOG_PATH}/archive/%d{yyyy-MM-dd}"/>
    <!--设置自定义pattern属性-->
    <property name="LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%X{traceId}| %-5level %logger{80} - %msg%n"/>

    <!--控制台输出日志-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <Pattern>${LOG_PATTERN}</Pattern>
        </encoder>
    </appender>

    <appender name="INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/info.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/info.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="WARN" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/warn.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>WARN</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/warn.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/error.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="CACHE-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/cache.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/cache.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="SPRING-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/spring.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/spring.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="MYBATIS-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/mybatis.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/mybatis.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="NETTY-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/netty.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/netty.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="FILTER-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/filter.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/filter.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="CONTROLLER-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/controller.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/controller.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="SERVICE-LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/service.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/service.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <springProfile name="prod,dev">
        <logger name="org.apache.ibatis" level="${LEVEL}" additivity="false">
            <appender-ref ref="MYBATIS-LOG"/>
        </logger>

        <logger name="org.mybatis.spring" level="${LEVEL}" additivity="false">
            <appender-ref ref="MYBATIS-LOG"/>
        </logger>

        <logger name="com.alibaba.druid" level="${LEVEL}" additivity="false">
            <appender-ref ref="MYBATIS-LOG"/>
        </logger>

        <logger name="io.netty" level="${LEVEL}" additivity="false">
            <appender-ref ref="NETTY-LOG"/>
        </logger>

        <logger name="org.springframework" level="${LEVEL}" additivity="false">
            <appender-ref ref="SPRING-LOG"/>
        </logger>

        <logger name="cache-log" level="${LEVEL}" additivity="false">
            <appender-ref ref="CACHE-LOG"/>
        </logger>
        <logger name="javax.cache" level="${LEVEL}" additivity="false">
            <appender-ref ref="CACHE-LOG"/>
        </logger>

        <logger name="com.example.demo.dao" level="${LEVEL}" additivity="false">
            <appender-ref ref="MYBATIS-LOG"/>
        </logger>

        <logger name="com.example.demo.manager" level="${LEVEL}" additivity="false">
            <appender-ref ref="SERVICE-LOG"/>
        </logger>

        <logger name="com.example.demo.service" level="${LEVEL}" additivity="false">
            <appender-ref ref="SERVICE-LOG"/>
        </logger>

        <logger name="com.example.demo.controller" level="${LEVEL}" additivity="false">
            <appender-ref ref="CONTROLLER-LOG"/>
        </logger>

        <logger name="com.example.demo.aop" level="${LEVEL}" additivity="false">
            <appender-ref ref="CONTROLLER-LOG"/>
        </logger>

        <root level="${ROOT_LEVEL}">
            <appender-ref ref="STDOUT"/>
        </root>
    </springProfile>

    <springProfile name="local">
        <root level="${ROOT_LEVEL}">
            <appender-ref ref="STDOUT"/>
        </root>
    </springProfile>
</configuration>
```