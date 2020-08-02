---
title: spring boot 入门：参数校验与错误捕获
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: c158ade3
date: 2019-03-17 02:01:00
updated: 2019-03-17 02:01:00
---

参数校验可直接使用 StringUtils 等工具类判断并返回错误响应，也可以借助 [hibernate-validator](http://hibernate.org/validator/documentation/)。首先需要添加 hibernate-validator 依赖。hibernate-validator 默认全量校验，通过以下配置类，可以使校验首次失败时即返回。

```java
@Configuration
public class ValidatorConfiguration {
    @Bean
    public Validator validator(){
        ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
                .configure()
                .addProperty( "hibernate.validator.fail_fast", "true" )// 校验失败，立即返回
                .buildValidatorFactory();
        Validator validator = validatorFactory.getValidator();

        return validator;
    }
}
```

### RequestBody 类入参校验

针对 requestBody 类入参，我们可以通过以下方式进行校验：

1. 实体类添加 @NotNull, @NotBlank, @Pattern 等注解，设定校验规则
2. 在控制器方法中使用 @Valid、BindingResult 标识参数

```java
@Data
public class User {
    private Integer id;

    @NotBlank(message="用户名不能为空")
    @Pattern(regexp="^[a-zA-Z]\\\\w{5,17}$",message="用户名须以英文字母起始，且不能包含中文或特殊字符")
    private String userName;

    @NotBlank(message="密码不能为空")
    @Pattern(regexp="^[a-zA-Z0-9]{6,16}$",message="密码须设为英文字母或数值，6到16位")
    private String password;

    @NotBlank(message="手机号不能为空")
    @Pattern(regexp="^1[0-9]{10}$",message="手机号格式不正确")
    private String mobile;

    @NotBlank(message="邮箱不能为空")
    @Pattern(regexp="^([a-z0-9A-Z]+[-|\\\\.]?)+[a-z0-9A-Z]@([a-z0-9A-Z]+(-[a-z0-9A-Z]+)?\\\\.)+[a-zA-Z]{2,}$",message="邮箱格式不正确")
    private String email;
}

@RestController
@RequestMapping("/user")
public class UserController extends BaseController {
    @PostMapping("/createUser")
    public Result createUser(@Valid User user, BindingResult result){
        if (result.hasErrors()) {
            ResultUtil.error(400, result.getAllErrors().get(0).getDefaultMessage());
        }
        return ResultUtil.success(user);
    }
}
```

### RequestParam 类参数

针对 RequestParam 类参数，不能简单地添加 @Valid 注解。我们需要如下处理：

1. ValidatorConfiguration 为 MethodValidationPostProcessor 实例设置 hibernate-validator 校验器。
2. controller 类中添加 @Validated 注解，查询参数前添加 @NotNull、@NotBlank、@Pattern 等注解。
3. 基于 @ControllerAdvice 捕获错误，获取错误提示并作为响应返回。

```java
@Configuration
public class ValidatorConfiguration {
    @Bean
    public MethodValidationPostProcessor methodValidationPostProcessor() {
        MethodValidationPostProcessor postProcessor = new MethodValidationPostProcessor();
        postProcessor.setValidator(validator());// 设置 validator 为快速失败返回模式
        return postProcessor;
    }

    @Bean
    public Validator validator(){
        ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
                .configure()
                .addProperty( "hibernate.validator.fail_fast", "true" )// 校验失败，立即返回
                .buildValidatorFactory();
        Validator validator = validatorFactory.getValidator();

        return validator;
    }
}


@RestController
@RequestMapping("/user")
@Validated
public class UserController extends BaseController {
    @GetMapping("/getUser")
    public Result getUser(
            @Pattern(regexp="^[1-9][0-9]*$",message="请输入正整数")
            @RequestParam(name = "id", required = true) String id){
        return ResultUtil.success(id);
    }
}

@ControllerAdvice
@RestController
@Slf4j
public class BaseController {
    @ExceptionHandler({Exception.class})
    public Result handleException(Exception e){
        log.info("error: "+e);
        return  ResultUtil.error(400, "error");
    }

    @ExceptionHandler({ValidationException.class})// 针对指定错误进行处理
    public Result handleValidationException(ValidationException exception) {
        String message = "params error";
        if(exception instanceof ConstraintViolationException){
            ConstraintViolationException exs = (ConstraintViolationException) exception;

            Set<ConstraintViolation<?>> violations = exs.getConstraintViolations();
            Iterator<ConstraintViolation<?>> constraintViolationIterator = violations.iterator();
            message = constraintViolationIterator.next().getMessage();
            log.info(message);
        }

        return ResultUtil.error(400, message);
    }
}
```

需要注意的是，当必填参数未传或参数类型转换失败，Servlet 机制就会将请求打回。

### 错误捕获

上文已点到，基于 @ControllerAdvice、@ExceptionHandler 可捕获全局错误并作出处理。在 web 应用的分层结构中，service 一般抛出异常，由 controller 层中的 BaseController 基类捕获错误并打印日志等。

```java
// 处理异常
@ControllerAdvice
@RestController
public class HandleForException {
    @ExceptionHandler({BusinessException.class})
    public String handleBusinessException(BusinessException e){
        // do something
    }

    @ExceptionHandler({DAOException.class})
    public String handleDAOException(DAOException e){
        // do something
    }
}
```

### hibernate-validator

#### 常用注解

* @Pattern(regex=, flags=)：正则表达式校验
* @NotNull、@Null：校验 null
* @NotEmpty：校验元素非空
* @NotBlank：校验 trim 后的字符串
* @AssertFalse、@AssertTrue：校验是否 false、true
* @DecimalMax(value=, inclusive=)、@DecimalMin(value=, inclusive=)：校验数值大小
* @Digits(integer=, fraction=)：校验整数、小数位数
* @Min(value=)、@Max(value=)、@Range(min=, max=)：最大值、最小值、范围校验
* @Negative、@NegativeOrZero、@Positive、@PositiveOrZero：正负值或零校验
* @Length(min=, max=)：校验字符串的长度
* @Size(min=, max=)：校验集合的大小
* @UniqueElements：校验集合包含的元素均唯一
* @Email：校验邮件
* @URL(protocol=, host=, port=, regexp=, flags=)：校验链接
* @Future、@FutureOrPresent、@Past、@PastOrPresent：校验日期是否将来或过去等
* @DurationMax(days=, hours=, minutes=, seconds=, millis=, nanos=, inclusive=)、@DurationMin(days=, hours=, minutes=, seconds=, millis=, nanos=, inclusive=)：校验时段
* @Valid：实体类可用于声明递归校验

#### 自定义校验器

以下为 @CheckCase 大小写格式注解的实现：

```java
@Target({ FIELD, METHOD, PARAMETER, ANNOTATION_TYPE, TYPE_USE })
@Retention(RUNTIME)// 在运行时可以通过反射获取
@Constraint(validatedBy = CheckCaseValidator.class)// 指定校验逻辑
@Documented
@Repeatable(List.class)
public @interface CheckCase {
    String message() default "{org.hibernate.validator.referenceguide.chapter06.CheckCase.message}";

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };

    CaseMode value();

    @Target({ FIELD, METHOD, PARAMETER, ANNOTATION_TYPE })
    @Retention(RUNTIME)
    @Documented
    @interface List {
        CheckCase[] value();
    }
}

public enum CaseMode {
    UPPER,
    LOWER;
}

public class CheckCaseValidator implements ConstraintValidator<CheckCase, String> {
    private CaseMode caseMode;

    @Override
    public void initialize(CheckCase constraintAnnotation) {
        this.caseMode = constraintAnnotation.value();
    }

    @Override
    public boolean isValid(String object, ConstraintValidatorContext constraintContext) {
        if ( object == null ) {
            return true;
        }

        if ( caseMode == CaseMode.UPPER ) {
            return object.equals( object.toUpperCase() );
        } else {
            return object.equals( object.toLowerCase() );
        }
    }
}
```