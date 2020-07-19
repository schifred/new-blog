---
title: spring boot 入门：控制器
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: '7981e696'
date: 2019-03-17 02:00:00
updated: 2019-03-17 02:00:00
---

首先来看一下 Spring MVC 提供的一些注解：

* @Controller：组合了 @Component，声明一个控制器。
* @RestController：组合注解，组合了 @Controller 和 @ResponseBody，将返回值放在 response 体内，以传递 json 数据。
* @ResponseBody：将 controller 返回值放在 response 体内。
* @RequestMapping：用于映射 Web 请求（访问路径和参数）、处理类和方法等。@RequestMapping 支持 Servlet 的 request 和 response 作为参数，也支持对 request 和 response 的媒体类型进行配置。典型形式如 
  * @RequestMapping(‘/users’) 或 @RequestMapping(value = ‘/users’, method = RequestMethod.GET)
  * @RequestMapping(path = “/users”, method = RequestMethod.PUT, consumes = “application/json”)
  * @RequestMapping(path = “/users”, method = RequestMethod.POST, produces = “application/json;charset=UTF-8”)
  * @RequestMapping(path = “/users”, method = RequestMethod.GET, params = “myParam=myValue”)
  * @RequestMapping(path = “/pets”, method = RequestMethod.GET, headers = “myHeader=myValue”)
  method 请求方法可以是 RequestMethod.GET, RequestMethod.POST, RequestMethod.PUT, RequestMethod.DELETE。
* @GetMapping，@PostMapping, @PutMapping, @DeleteMapping, @PatchMapping：简便形式映射 get, post 等请求。
* @PathVariable：用来接收路径参数，如 users/{id} 路径中的 id。
* @RequestParam：用来接收 get, post 请求中查询路径中的参数，可处理 Content-Type: application/x-www-form-urlencoded 编码格式的数据，形式如 @RequestParam(value=”id”) int id。
* @RequestBody：用来接收 json 格式的数据，可将其转换成对应的数据类型，一般用于处理非 Content-Type: application/x-www-form-urlencoded 编码格式的数据，不能处理 get 请求。
* @ModelAttribute：将参数绑定到 Model 对象上。
* @RequestHeader：获取消息头中的内容。
* @CookieValue：用来接收 cookie 的值。
* @SessionAttributes：用来将指定实体类的数据存储到 session，一般用在类级别。
* @SessionAttribute：用来接收已经存储的 session，一般用在方法级别。
* @RequestAttribute：用来接收请求参数，一般用在过滤器或拦截器上。
* HttpServletRequest：获取请求内容。

针对请求头 ContentType 指定的编码格式，有如下三种情况：

1. application/x-www-form-urlencoded：@RequestParam, @ModelAttribute, @RequestBody 均可以处理。
multipart/form-data：不能使用 @RequestBody 处理（form表单中有文件时，必须指定 enctype 属性为 multipart/form-data，意为以二进制流的形式传输文件）。
2. application/json、application/xml等：必须使用 @RequestBody 处理。
3. 针对 ajax 请求，首先制作 ResultUtil 统一处理返回值；然后使用注解编写 controller，代码如下：

```java
// 使用 ResultUtil 统一处理返回值
public class Result<T> {
    private Integer code;
    private String msg;
    private Boolean success;
    private T data;

    public Integer getCode(){
        return code;
    }

    public void setCode(Integer code){
        this.code = code;
    }

    public String getMsg(){
        return msg;
    }

    public void setMsg(String msg){
        this.msg = msg;
    }

    public Integer getSuccess(){
        return success;
    }

    public void setSuccess(Boolean success){
        this.success = success;
    }

    public T getData(){
        return data;
    }

    public void setData(T data){
        this.data = data;
    }
}

public class ResultUtil {
    public static Result success(Object object){
        Result result = new Result();
        result.setCode(200);
        result.setMsg("success");
        result.setSuccess(true);
        result.setData(object);
        return result;
    }

    public static Result success(){
        return success(null);
    }

    public static Result error(Integer code, String msg){
        Result result = new Result();
        result.setCode(code);
        result.setMsg(msg);
        result.setSuccess(false);
        result.setData(null);
        return result;
    }
}

// controller
@RequestMapping("/request")
@RestController
public class RequestController {
    @RequestMapping(value = "/pathVariableTest/{id}")
    public Result pathVariableTest(@PathVariable int id){
        return ResultUtil.success();
    }

    @RequestMapping(value = "/requestParamTest")
    public Result requestParamTest(@RequestParam(value = "id") int id){
        return ResultUtil.success();
    }

    @RequestMapping(value = "/requestBodyTest", method = RequestMethod.POST)
    public Result requestBodyTest(@RequestBody User user){
        return ResultUtil.success(user);
    }

    @RequestMapping(value = "/modelAttributeTest", method = RequestMethod.POST)
    public Result modelAttributeTest(@ModelAttribute User user){
        return ResultUtil.success(user);
    }

    @RequestMapping(value = "httpServletRequestTest", method = RequestMethod.POST)
    public Result httpServletRequestTest(HttpServletRequest request){
        return ResultUtil.success(request);
    }
}
```

### 页面渲染

针对页面，Spring Boot 支持使用 FreeMarker, Groovy, Thymeleaf, Velocity, Mustache 作为模板引擎，推荐使用 Thymeleaf，因为其提供了完美的 Spring MVC 的支持。编写方法为：pom.xml 文件添加 thymeleaf 依赖，制作 controller 直接返回页面模板在 resources/templates 中的位置，添加 html 页面。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

```java
@Controller
public class ThymeleafController {
    @RequestMapping("/")
    public String index(){
        return "index";
    }
}
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>demo</title>
</head>
<body>
    just a demo
</body>
</html>
```

### 参考

[@RequestParam、@RequestBody和@ModelAttribute区别](https://www.cnblogs.com/zeroingToOne/p/8992746.html)
[@SessionAttributes与@SessionAttribute会话数据控制](http://www.imooc.com/article/details/id/262098)
[HttpServletRequest对象](https://www.cnblogs.com/xdp-gacl/p/3798347.html)
[利用 Spring Boot 设计风格良好的Restful API及错误响应](https://www.jianshu.com/p/d6424d98b02e)
[springboot使用hibernate validator校验](https://www.cnblogs.com/mr-yang-localhost/p/7812038.html)