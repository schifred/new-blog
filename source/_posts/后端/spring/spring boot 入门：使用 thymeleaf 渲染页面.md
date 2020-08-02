---
title: spring boot 入门：使用 thymeleaf 渲染页面
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 817b8053
date: 2019-03-17 02:05:00
updated: 2019-03-17 02:05:00
---

Spring Boot 支持使用 FreeMarker, Groovy, Thymeleaf, Velocity, Mustache 作为模板引擎，推荐使用 Thymeleaf，因为它提供了完美的 Spring MVC 支持。我们可以按以下方式使用 Thymeleaf 模板引擎：

1. 添加 spring-boot-starter-thymeleaf 依赖
2. 制作 html 页面
3. 制作 controller 直接返回页面模板在 resources/templates 目录中的路径（resources/templates、view/templates 是 spring boot 的默认模板存放位置）

```java
@Controller
public class PageController {
  @GetMapping("/")
  public String getIndex(Model model){
    return "index";
  }
}
```

### 注入动态数据

在方法中声明参数为 Model 类，通过 Model#addAttribute 方法就可以为页面注入动态数据。

```java
@Controller
public class PageController {
    @GetMapping("/")
    public String getIndex(Model model){
        String env = System.getProperty("spring.profiles.active");
        model.addAttribute("env",env);
        return "index";
    }
}
```

前端应用通常是单体应用，我们通常只需要在 index.html 指定加载的脚本就行。我们仅需要知道以下语法，不必对 thymeleaf 过于熟悉。

* ${someVariable}：变量表达式。在 {} 中使用 #someVariable 可获取 thymeleaf 注入的日期、session、上下文、国际化等信息。
* @{someUrlWithVaraible}：网址表达式。

在标签中，通过 "th:text" 或 "th:href" 等属性对页面进行设值。"th:href" 属于 thymeleaf 使用自定义属性包装 html 节点属性的一个示例。"th:with" 属性可定义局部变量。

需要说明的是，使用 thymeleaf 获取动态数据时，需要指定 xml 的命名空间，这样才能使用 thymeleaf 的自定义标签。

```html
<!DOCTYPE html>
<!-- 指定命名空间 -->
<html lang="en" xmlns:th="http://www.thymeleaf.org/" th:with="ver=${#dates.createNow().time}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <link rel="stylesheet" th:href="@{https://myCssUrlPrefix/{env}/umi.css(env=${env})}">
    <title>spring boot demo</title>
</head>
<body>
    <div id="root">页面加载中</div>
    <script th:src="@{https://myJsUrlPrefix/{env}/umi.js(env=${env})}"></script>
</body>
</html>
```

更多内容可戳 [SpringBoot【Thymeleaf篇】](https://zhuanlan.zhihu.com/p/103089477)

### 防止 csrf 攻击

将防止 csrf 攻击归入这一小节的原因在于，实现上我们首先将 csrf token 传入 html 页面，然后在前端请求头中添加这个 token。简要步骤如下：

1. 添加 spring-security-web 依赖。
2. 在过滤器配置类中注册 CsrfFilter 实例。CsrfFilter 实例会通过 request.setAttribute 方法将 _csrf 写入到页面模板中。
3. html 页面中获取 csrf token 并写入 window 全局变量。
4. 前端发送请求时携带 csrf token。

```java

@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean registerCsrfFilter() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new CsrfFilter(new HttpSessionCsrfTokenRepository()));
        registration.addUrlPatterns("/*");
        return registration;
    }
}
```

```html

<!DOCTYPE html>
<!-- 指定命名空间 -->
<html lang="en" xmlns:th="http://www.thymeleaf.org/" th:with="ver=${#dates.createNow().time}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <link rel="stylesheet" th:href="@{https://myCssUrlPrefix/{env}/umi.css(env=${env})}">
    <title>spring boot demo</title>
    <script th:inline="javascript">
        window._csrf = [[${_csrf}]];// { token, parameterName, headerName }
    </script>
</head>
<body>
    <div id="root">页面加载中</div>
    <script th:src="@{https://myJsUrlPrefix/{env}/umi.js(env=${env})}"></script>
</body>
</html>
```