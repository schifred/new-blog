---
title: spring boot 入门：aop 切面编程
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: ad1e9018
date: 2019-03-17 02:20:00
updated: 2019-03-17 02:20:00
---

面向切面编程把横切关注点和业务逻辑分离，以便为横切关注点单独制作模块。面向切面编程适用于日志、安全和事务管理等场景。spring 中的 AOP 编程发挥了声明式编程的特点，即使用 @Before、@After 等注解在某方法或某类方法的特定位置织入代码（织入的代码被称为建言）。

一般而言，spring 使用 execution 表达式定位方法，如 @Before("execution(public * com.example.demo.controller..*.*(..))") 直接定位到 controller 层中所有方法。除了为一类方法织入代码外，我们也可以通过自定义注解的方式间接定位需要织入代码的方法。

* @Aspect：声明这是一个切面，约定需要织入的代码和织入的方式。 
* @After, @AfterReturning, @AfterThrowing, @Before, @Around：声明在方法前或者后织入代码。
* @PointCut：声明一个切点，即在哪个方法上织入代码。通常在间接定位形式中使用。

本文仅简要地介绍 AOP 编程的方式。使用切面前须引入 spring-boot-starter-aop 依赖。

### 直接定位

1. 使用 @Aspect, @Component 注解声明一个切面。
2. 切面中使用 @After, @AfterReturning, @AfterThrowing, @Before, @Around 约定建言。

如以下代码就会在 controller 层所有方法执行前打印日志。

```java
@Slf4j
@Aspect
@Component
@Order(1)
public class LogAspect {
    //申明一个切点 里面是 execution表达式
    @Pointcut("@annotation(com.example.demo.aop.Log)")
    private void controllerAspect(){}

    @Before("execution(public * com.example.demo.controller..*.*(..))")
    public void methodBefore(JoinPoint joinPoint){
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = requestAttributes.getRequest();
        MethodSignature signature = (MethodSignature)joinPoint.getSignature();

        String[] paramNames = signature.getParameterNames();
        Map<String, String> paramsMap = new HashMap<>(paramNames.length);
        Object[] args = joinPoint.getArgs();
        for (int i = 0; i < paramNames.length; i++){
            Object arg = args[i];
            if (arg instanceof HttpServletRequest){

            // 响应被 JSON.toJSONString 处理后，图片校验码接口报错
            // getOutputStream() has already been called for this response
            } else if (arg instanceof HttpServletResponse) {

            } else {
                paramsMap.put(paramNames[i], JSON.toJSONString(arg));
            }
        }

        log.info("===============请求内容===============");
        log.info("请求地址: {}", request.getRequestURL().toString());
        log.info("请求方式: {}", request.getMethod());
        log.info("请求类方法: {}", signature);
        log.info("请求类方法参数: {}", paramsMap);
        log.info("===============请求内容===============");
    }

    @AfterReturning("execution(public * com.example.demo.controller..*.*(..))")
    public void methodAfterReturing(Object o){
        log.info("===============响应内容===============");
        log.info("响应内容: ", JSON.toJSONString(o));
        log.info("===============响应内容===============");
    }
}
```

### 间接定位

1. 声明辅助类，通常是一个注解。
2. 使用 @Aspect, @Component 注解声明一个切面。
3. 切面中使用 @Pointcut 将辅助类定义为连接点。
4. 切面中使用 @After, @AfterReturning, @AfterThrowing, @Before, @Around 针对辅助类约定建言。
5. 在 controller, service 的方法中添加辅助类注解或显式调用辅助类的方法。

如以下代码会在添加了 @Log 注解的方法上打印日志。

```java
// 辅助类
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log {
    String log();
}

@Slf4j
@Aspect
@Component
@Order(1)
public class LogAspect {
    //申明一个切点 里面是 execution 表达式
    @Pointcut("@annotation(com.example.demo.aop.Log)")
    private void controllerAspect(){}

    //在方法执行完结后打印返回内容
    @AfterReturning(returning = "o",pointcut = "controllerAspect()")
    public void methodAfterReturing(Object o){
        log.info("--------------返回内容----------------");
        log.info("Response内容:" + JSON.toJSONString(o));
        log.info("--------------返回内容----------------");
    }
}

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
