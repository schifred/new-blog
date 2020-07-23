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

如 [spring boot 入门：搭建项目](/archives/b16f5632/) 篇所说，使用 @Controller 注解的类可以被自动扫描到并加载为 Bean 实例。@Controller 由 Spring MVC 提供，它组合了 @Component，相应的控制器实例也会在 Spring MVC 中用于完成匹配请求、解析参数、获取响应等行为。

### 匹配请求

在方法或类上添加 @RequestMapping 注解，表明该方法或类在匹配到特定请求的路径、方法以及头信息时，就会被触发执行。比如在类上添加 @RequestMapping(‘/users’) 注解，表明该类在用户访问 localhost:8080/users 时生效。

请求方法默认为 GET 请求，可以通过 @RequestMapping(path = “/users”, method = RequestMethod.POST) 配置为 POST 请求。method 值支持 RequestMethod.GET, RequestMethod.POST, RequestMethod.PUT, RequestMethod.DELETE 等。

除了 path, method 外，其他注解属性包含 consumes 请求的媒体类型、produces 响应的媒体类型、headers 请求头、charset 字符编码、params 参数格式等。使用如 @RequestMapping(path = “/users”, method = RequestMethod.GET, params = “myParam=myValue”)。

基于 @RequestMapping 注解，@GetMapping，@PostMapping, @PutMapping, @DeleteMapping, @PatchMapping 等注解可用于便利地映射 GET、POST 等请求。

### 获取参数

以下列出了四类参数，以及它们分别需要使用什么注解：

* 查询参数：指 url 路径上 ? 后的内容，如 paramName=paramValue，可使用 @RequestParam 注解接收，如 @RequestParam(value=”paramName”) string paramName 获取
* 请求体：指 request body，一般在 post 请求中使用，content-type 可以是 application/json、application/xml 等。可使用 @RequestBody User user 获取
* 表单数据：指 content-type 为 application/x-www-form-urlencoded 的表单提交数据，可使用 @ModelAttribute, @RequestBody 注解接收
* 路径变量：指 users/{id} 路径中的 id，可以用 @PathVariable("id") int id 获取

除此之外，以下注解也是可用的：

* @RequestAttribute：用来接收请求参数，一般用在过滤器或拦截器上。
* @RequestHeader：获取消息头中的内容。
* @CookieValue：用来接收 cookie 的值。
* @SessionAttributes：用来将指定实体类的数据存储到 session，一般用在类级别。
* @SessionAttribute：用来接收已经存储的 session，一般用在方法级别。

除了注解，在参数直接使用 HttpServletRequest 标记参数类型可用于获取请求对象，HttpServletResponse 可用于获取请求对象，这通常用来作特殊处理，比如使用 HttpServletResponse 实例返回图片等。

### 获取响应

对于 ajax 请求，在方法上添加 @ResponseBody 注解可以将 json 数据写入响应对象中。如果在类上添加 @RestController，就不必在方法上添加 @ResponseBody 注解。因为 @RestController 组合了 @Controller 和 @ResponseBody 注解。

如上文所述，对于需要特殊处理的响应，可以直接操纵 HttpServletResponse 实例。

### 代码示例

#### ajax 请求

我们一般需要创建如 ResultUtils 工具类用于生成成功或失败的响应内容，代码如下：

```java
import com.alibaba.fastjson.JSON;

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

public class ResultUtils {
    public Result success(Object object){
        Result result = new Result();
        result.setCode(200);
        result.setMsg("success");
        result.setSuccess(true);
        result.setData(object);
        return result;
    }

    public Result success(Object obj){
        if (obj instanceof String){
            return success(null, (String) obj);
        }else{
            return success(obj, "success");
        }
    }

    public Result success(Object obj, String msg){
        Result result = new Result();
        result.setCode(200);
        result.setMsg(msg);
        result.setSuccess(true);
        result.setData(object);
        return result;
    }

    public Result error(Integer code){
        Result result = new Result();
        result.setCode(code);
        result.setMsg("error");
        result.setSuccess(false);
        result.setData(null);
        return result;
    }

    public Result error(Object obj){
        if (obj instanceof String){
            return error(null, (String) obj);
        }else{
            return error(obj, "error");
        }
    }

    public Result error(Integer code, String msg){
        Result result = new Result();
        result.setCode(code);
        result.setMsg(msg);
        result.setSuccess(false);
        result.setData(null);
        return result;
    }
}
```

基于 ResultUtils，我们可编写如下 controller：

```java
// controller
@RequestMapping("/request")
@RestController
public class RequestController {
    @RequestMapping(value = "/pathVariableTest/{id}")
    public Result pathVariableTest(@PathVariable int id){
        return ResultUtils.success();
    }

    @RequestMapping(value = "/requestParamTest")
    public Result requestParamTest(@RequestParam(value = "id") int id){
        return ResultUtils.success();
    }

    @RequestMapping(value = "/requestBodyTest", method = RequestMethod.POST)
    public Result requestBodyTest(@RequestBody User user){
        return ResultUtils.success(user);
    }

    @RequestMapping(value = "/modelAttributeTest", method = RequestMethod.POST)
    public Result modelAttributeTest(@ModelAttribute User user){
        return ResultUtils.success(user);
    }

    @RequestMapping(value = "httpServletRequestTest", method = RequestMethod.POST)
    public Result httpServletRequestTest(HttpServletRequest request){
        return ResultUtils.success(request);
    }
}
```

#### 参数校验

参数校验可直接使用 StringUtils 等工具类判断并返回错误响应，也可以借助 hibernate-validator。以下是使用示例：

1. 添加 hibernate-validator 依赖
2. 实体类添加 @NotNull 等注解，设定校验规则
3. 在控制器方法中使用 @Valid、BindingResult 标识参数

```java
public class Car {
  @NotNull
  private String manufacturer;

  @NotNull
  @Size(min = 2, max = 14)
  private String licensePlate;

  @Min(2)
  private int seatCount;
}

@RestController
public class UserController {
  @GetMapping("/getCar")
  @ResponseBody
  public Result<Car> insertUser(@Valid Car car, BindingResult result) {
    if (result.hasErrors()) {
      ResultUtils.error(
        ResponseConstants.PARAM_ERROR.getCode(), 
        result.getAllErrors()[0].getDefaultMessage()
      );
    }
    return ResultUtils.success(car);
  }
}
```

#### 错误捕获

使用 @ExceptionHandler 能捕获同一个类上抛出的错误，使用时需要指定错误类型。@ControllerAdvice 可用于捕获全局错误，这样可以打印日志等。

在 web 应用的分层结构，service 一般抛出异常，controller 层用于捕获错误并作出处理。这时候，就可以使用 @ControllerAdvice、@ExceptionHandler 作统一处理。

```java
// 处理异常
@ControllerAdvice
public class HandleForException {
    @ExceptionHandler({ArithmeticException.class})// 针对指定错误进行处理
    public String testArithmeticException(Exception e){
        System.out.println("ArithmeticException:"+e);
        return "error";
    }
}
```

#### 页面渲染

Spring Boot 支持使用 FreeMarker, Groovy, Thymeleaf, Velocity, Mustache 作为模板引擎，推荐使用 Thymeleaf，因为其提供了完美的 Spring MVC 支持。

1. 在 pom.xml 文件添加 spring-boot-starter-thymeleaf 依赖
2. 制作 html 页面
3. 制作 controller 直接返回页面模板在 resources/templates 中的位置
4. Model 可用于页面注入动态数据

```java
@Controller
public class ThymeleafController {
    @RequestMapping("/")
    public String index(Model model){
        model.addAttribute("timestamp", new Date().getTime());
        return "index";
    }
}
```

#### 文件上传

使用 @RequestParam、MultipartFile 可接收请求中的文件对象。

```java
@PostMapping("/uploadFile")
public Result<FileDTO> uploadFile(@RequestParam("file") MultipartFile file){
  FileDTO fileDTO = storageService.store(file);// 实现存储 service
  return Result.createSuccessResult(fileDTO);
}
```

更多内容可戳 [Uploading Files 文档](https://spring.io/guides/gs/uploading-files/)。