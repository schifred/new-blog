---
title: spring boot 入门：logback 日志
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 89954b97
date: 2019-03-17 01:10:00
updated: 2019-03-17 01:10:00
---

[spring-boot-starter-web](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web) 整合了 [spring-boot-starter-logging](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-logging)。在 maven 官网中，我们可以看到 spring-boot-starter-logging 整合了 [logback-classic](https://mvnrepository.com/artifact/ch.qos.logback/logback-classic)、[log4j-to-slf4j](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-to-slf4j)、[jul-to-slf4j](https://mvnrepository.com/artifact/org.slf4j/jul-to-slf4j)。一般集成 logback 服务也需要 logback-classic、log4j-to-slf4j、jul-to-slf4j 三个包。

log4j-over-slf4j 通过代理将系统中所有 log4j 日志（含第三方库的）路由到 slf4j上。jcl-over-slf4 将 apache commons logging 路由到 slf4j上。jul-over-slf4 将 java.util.logging 路由到 slf4j 上。更多内容可戳 [log4j-over-slf4j 工作原理详解](https://blog.csdn.net/john1337/article/details/76152906)、[logback 详解](https://blog.csdn.net/Sadlay/article/details/88732271)。

需要注意的是，当添加 druid 数据源后，我们会看见在项目启动过程中存在 [Failed to bind properties under 'spring.datasource' to javax.sql.DataSource](https://blog.csdn.net/xingkongtianma01/article/details/81624313) 报错，此时添加 log4j-over-slf4j 即可。

### spring boot 默认日志

因为 Spring Boot 默认使用了 logback 提供日志服务，我们可以直接在项目中使用如下形式打印日志：

```java
@Service
class TestServiceImpl implements TestService {
  private static final Logger log = LoggerFactory.getLogger(TestService.class);

  public void doTest(){
    // do something
    log.info("print error");
  }
}
```

日志级别从高到低分别为：TRACE < DEBUG < INFO < WARN < ERROR < FATAL。日志级别可以在 application.yml 中设置：

```yml
logging:
  level:
    root: debug
    # com.demo: info # 设定工程包的日志输出级别
  # pattern:
  #   console: "%d -%msg%n"    # 日志格式
  # path: /log/data/demo       # 日志文件,绝对路径或相对路径
```

### 使用 lombok 简化编码

以上每次在使用日志前，需要调用 LoggerFactory.getLogger，基于 [lombok](https://mvnrepository.com/artifact/org.projectlombok/lombok/1.18.12) 提供的 @Slf4j 注解可以将 log 注入到类中，这样就可以简化编码。

```java
@Slf4j
@Service
class TestServiceImpl implements TestService {
  public void doTest(){
    // do something
    log.info("print error");
  }
}
```

对于 log4j2 日志，lombok 提供了 @Log4j2 注解。

### logback 配置

以下配置用于说明 logback.xml 配置文件各标签的意义。完整配置可参考 [spring boot 入门：配置文件](/archives/6fd0dc6f/) 篇。更多可戳 [logback 官网](http://logback.qos.ch/demo.html)、[logback-demo](https://github.com/qos-ch/logback-demo)、[什么是 Appender](https://www.cnblogs.com/yw0219/p/9361040.html)、[如何正确配置logback](https://zhuanlan.zhihu.com/p/100713439)。

```yml
# application.yml：
logging:
  config: classpath:logback-config.xml
  file:
    path: /Users/alfred # mac 下根路径创建 data 目录需要权限
  level:
    all: debug
    root: debug
spring:
  application:
    name: demo # logback-config.xml 基于应用名创建日志文件夹
```

```xml
<!-- logback-config.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<configuration>
    <!-- 集成 spring boot 默认日志配置 -->
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
    <!-- 设置日志格式 -->
    <property name="LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%X{traceId}| %-5level %logger{80} - %msg%n"/>

    <!-- 基于 ConsoleAppender 将日志打印到控制台，标识符为 STDOUT -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <Pattern>${LOG_PATTERN}</Pattern>
        </encoder>
    </appender>

    <!-- 基于 RollingFileAppender 打印日志文件，过滤 info 日志 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/info.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <!-- 日志文件策略配置 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ARCHIVE_LOG_PATH}/info.%d{yyyy-MM-dd}.log.%i</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- 标识符为 CONTROLLER-LOG，缓存 controller 层日志 -->
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

    <!-- 基于 springProfile 对线上、开发环境作处理 -->
    <springProfile name="prod,dev">
        <!-- 使用标识符为 CONTROLLER-LOG 的 appender 打印 com.example.demo.controller 层日志 -->
        <logger name="com.example.demo.controller" level="${LEVEL}" additivity="false">
            <appender-ref ref="CONTROLLER-LOG"/>
        </logger>

        <!-- 默认标识符为 STDOUT 的 appender，打印到控制台 -->
        <root level="${ROOT_LEVEL}">
            <appender-ref ref="STDOUT"/>
        </root>
    </springProfile>

    <!-- 非 prod、dev 环境，以 STDOUT、FILE 形式打印日志 -->
    <root>
      <level value="DEBUG"/>
      <appender-ref ref="STDOUT"/>
      <appender-ref ref="FILE"/>
    </root>
</configuration>
```