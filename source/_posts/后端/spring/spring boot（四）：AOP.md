---
title: spring boot（四）：AOP
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 63b17f08
date: 2019-03-17 03:00:00
updated: 2019-03-17 03:00:00
---

面向切面编程能把横切关注点和业务逻辑相互分离开，以便于为横切关注点单独制作模块。通常，横切关注点包括日志、安全和事务管理等通用内容。以比方论，面向切面编程就是在一个处理过程中定位一个切点，然后在这个切点前后添加一个或多个动作；其实现依赖于 Java 编程的同步性。因此，面向切面编程需要包含切点定位、动作定义、动作相对于切点的执行位置等。在一个处理过程中添加动作本可以借助显式调用函数的方式实现，但这样处理过程就会与一个或多个特定的动作高度耦合，且当另一个处理过程需要添加这一个或多个特定的动作时，仍需要显式调用。如果在处理过程中插入一个空跑的函数声明这是个切点，然后再使用代理封装该函数，就可以在切点前后以松耦合的方式执行特定动作了。特定动作和待封装的函数可能是一对多的关系。spring AOP 以切面（Aspect）为视角，首先以切点（PointCut）声明待封装的函数在哪个位置，其次使用 @After, @Before, @Around 注解声明在切点前后需要执行哪些动作。使用 @PointCut 定义切点是不必要的，它只是一种语法糖；因为在 @After 注解后可直接使用 execution 表达式锁定待封装函数的位置，同时 execution 表达式也能定位到多个待封装函数。特别的，当 execution 表达式定位 controller 时，就可以对特定或所有的 controller 进行拦截，而无需插入在 controller 插入定义为连接点（JoinPoint）的动作，controller 就作为连接点。想要实现这一过程，spring 容器在创建 controller 等定义了连接点的实例时，就必须知道哪些是 Aspect，且这些 Aspect 都需要注明为 Bean 以实例化，然后再使用这些实例去封装定义了连接点的实例。基于上述，AOP 有三种使用方式：

1. execution 表达式直接定位 controller, service 的方法。
2. execution 表达式定位辅助类，再通过辅助类将 controller, service 的方法声明为连接点。
3. execution 表达式定位辅助类，在 controller, service 中显式调用辅助类中作为连接点的方法。

编码过程如下：

1. 使用 @Aspect, @Bean 注解声明一个切面。
2. 在切面中使用 @After, @AfterReturning, @AfterThrowing, @Before, @Around 注解定义一个建言（advice）。或者以 execution 表达式定位连接点的位置；或者在切面中 @PointCut 定义切点，并在 @After, @AfterReturning, @AfterThrowing, @Before, @Around 注解中使用该切点作为参数。
3. controller, service 的方法作为连接点，无需额外编程；辅助类的方法作为连接点，或者在 controller, service 的方法上添加注解，或者在 controller, service 的方法中显示调用辅助类的方法。

本节以 AOP 统一日志处理为例，说明其使用过程：

1. pom.xml 添加 aop, log4j 依赖。

```xml
<!-- 禁用默认日志模块 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j</artifactId>
</dependency>
```

2. resources 目录中添加 log4j.properties 配置文件。

```bash
log4j.rootLogger=INFO, CONSOLE, ROLLING_FILE, DAILY_ROLLING_FILE

log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=<%d>[%5p] %m - %c%n

log4j.appender.ROLLING_FILE=org.apache.log4j.RollingFileAppender
log4j.appender.ROLLING_FILE.File=./logs/client.log
log4j.appender.ROLLING_FILE.Append=true
log4j.appender.ROLLING_FILE.MaxFileSize=20000KB
log4j.appender.ROLLING_FILE.MaxBackupIndex=100
log4j.appender.ROLLING_FILE.layout=org.apache.log4j.PatternLayout

#log4j.appender.ROLLING_FILE.layout.ConversionPattern=<%d>[%5p] %c - %m%n
log4j.appender.ROLLING_FILE.layout.ConversionPattern=%d %c [%t] (%F:%L) %-5p --> %m%n

log4j.appender.DAILY_ROLLING_FILE=org.apache.log4j.DailyRollingFileAppender
log4j.appender.DAILY_ROLLING_FILE.File=./logs/client
log4j.appender.DAILY_ROLLING_FILE.DatePattern='.'yyyy-MM-dd'.log'
log4j.appender.DAILY_ROLLING_FILE.Append=true
log4j.appender.DAILY_ROLLING_FILE.layout=org.apache.log4j.PatternLayout
log4j.appender.DAILY_ROLLING_FILE.layout.ConversionPattern=%d %c [%t] (%F:%L) %-5p --> %m%n
```

3. 制作 Aspect 类。

```java
@Aspect
@Component
@Order(1)
public class LogAspect {
    private Logger log = Logger.getLogger(LogAspect.class);

    //申明一个切点 里面是 execution表达式
    @Pointcut("@annotation(com.example.demo.aop.Log)")
    private void controllerAspect(){}

    //请求method前打印内容
    @Before("execution(public * com.example.demo.controller..*.*(..))")
    public void methodBefore(JoinPoint joinPoint){
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = requestAttributes.getRequest();

        //打印请求内容
        log.info("===============请求内容===============");
        log.info("请求地址:"+request.getRequestURL().toString());
        log.info("请求方式:"+request.getMethod());
        log.info("请求类方法:"+joinPoint.getSignature());
        log.info("请求类方法参数:"+ Arrays.toString(joinPoint.getArgs()));
        log.info("===============请求内容===============");
    }


    //在方法执行完结后打印返回内容
    @AfterReturning(returning = "o",pointcut = "controllerAspect()")
    public void methodAfterReturing(Object o){
        log.info("--------------返回内容----------------");
        log.info("Response内容:" + JSON.toJSONString(o));
        log.info("--------------返回内容----------------");
    }
}
```

4. 声明辅助类。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log {
    String log();
}
```

5. 被拦截的处理过程。

```java
@RestController
@RequestMapping(value = "/aop")
public class AopController {
    @Log(log = "注解式拦截的方法")
    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public Result getTest(){
        return ResultUtil.success();
    }
}
```

![image](aop.png)

### 参考

[Spring Boot项目中使用log4j](https://blog.csdn.net/xiaoxiaoyusheng2012/article/details/79486784)
[annotation(@Retention@Target)详解](https://www.cnblogs.com/gmq-sh/p/4798194.html)