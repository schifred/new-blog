---
title: spring boot 进阶：hibernate validator
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring,hibernate
abbrlink: f0e63bcb
date: 2020-05-02 06:00:00
updated: 2020-05-02 06:00:00
---

从表现层到持久层都需要作数据校验，为了避免在各层实现校验函数的繁琐、性能消耗，[hibernate-validator](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#preface) 依循 Jakarta Bean Validation 2.0 规范将校验规则绑定在领域模型 —— 实体类上（通过注解或 xml 文件的形式为实体类添加元数据描述）。

### 一般校验

1. pom.xml 文件添加 hibernate-validator 依赖。
2. 实体类添加 @NotNull 等注解，设定校验规则，参考 [Applying constraints](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-gettingstarted-createmodel)。
3. 显式调用 Validation.buildDefaultValidatorFactory 等进行校验，参考 [Validating constraints](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#_validating_constraints)。

```java
public class Car {
  @NotNull
  private String manufacturer;

  @NotNull
  @Size(min = 2, max = 14)
  private String licensePlate;

  @Min(2)
  private int seatCount;

  public Car(String manufacturer, String licencePlate, int seatCount) {
    this.manufacturer = manufacturer;
    this.licensePlate = licencePlate;
    this.seatCount = seatCount;
  }
}

ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
validator = factory.getValidator();

// 1
Car car = new Car("Morris", "D", 4);
Set<ConstraintViolation<Car>> constraintViolations =
  validator.validate( car );
assertEquals(1, constraintViolations.size() );
assertEquals(
  "size must be between 2 and 14",
  constraintViolations.iterator().next().getMessage()
);

// 2
Car car = new Car( null, true );
Set<ConstraintViolation<Car>> constraintViolations = validator.validateProperty(
  car,
  "manufacturer"
);
assertEquals( 1, constraintViolations.size() );
assertEquals( "must not be null", constraintViolations.iterator().next().getMessage() );

// 3
Set<ConstraintViolation<Car>> constraintViolations = validator.validateValue(
  Car.class,
  "manufacturer",
  null
);
assertEquals( 1, constraintViolations.size() );
assertEquals( "must not be null", constraintViolations.iterator().next().getMessage() );
```

### 常用注解

hibernate-validator 的校验条件可以放在属性、方法、Map 或 List 成员、类等。校验条件主要包含：

* @NotNull、@Null：校验 null
* @NotEmpty：校验元素非空
* @NotBlank：校验 trim 后的字符串
* @AssertFalse、@AssertTrue：校验是否 false、true
* @DecimalMax(value=, inclusive=)、@DecimalMin(value=, inclusive=)：校验数值大小
* @Digits(integer=, fraction=)：校验整数、小数位数
* @Min(value=)、@Max(value=)、@Range(min=, max=)：最大值、最小值、范围校验
* @Negative、@NegativeOrZero、@Positive、@PositiveOrZero：正负值或零校验
* @Future、@FutureOrPresent、@Past、@PastOrPresent：校验日期是否将来或过去等
* @DurationMax(days=, hours=, minutes=, seconds=, millis=, nanos=, inclusive=)、@DurationMin(days=, hours=, minutes=, seconds=, millis=, nanos=, inclusive=)：校验时段
* @Length(min=, max=)：校验字符串的长度
* @Size(min=, max=)：校验集合的大小
* @UniqueElements：校验集合包含的元素均唯一
* @Currency(value=)：校验货币单位
* @Pattern(regex=, flags=)：正则表达式校验
* @Email：校验邮件
* @URL(protocol=, host=, port=, regexp=, flags=)：校验链接
* @CreditCardNumber(ignoreNonDigitCharacters=)：信用卡校验

### spring-boot

```java
@RestController
public class UserController {
  @GetMapping("/getCar")
  @ResponseBody
  public Result<Car> insertUser(@Valid Car car, BindingResult result) {
    if (result.hasErrors()) {
      Result.createFailResult(
        String.valueOf(ResponseConstants.PARAM_ERROR.getCode()), 
        result.getAllErrors()[0].getDefaultMessage()
      );
    }
    return Result.createSuccessResult(car);
  }
}
```

### 自定义校验器

自定义校验器通过以下步骤制作：

1. 创建约束条件注解。
2. 实现 validator。
3. 定义默认的错误提示。

以下是官方提供的自定义字符串大小写校验器示例：

```java
@Target({ FIELD, METHOD, PARAMETER, ANNOTATION_TYPE, TYPE_USE })
@Retention(RUNTIME)// 在运行时可以通过反射获取
@Constraint(validatedBy = CheckCaseValidator.class)// 指定校验逻辑
@Documented
@Repeatable(List.class)
public @interface CheckCase {
    // Jakarta Bean Validation API 规范，message 默认错误提示
    String message() default "{org.hibernate.validator.referenceguide.chapter06.CheckCase.message}";

    // Jakarta Bean Validation API 规范，message 默认错误提示
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

### 参考

[@Validated和@Valid区别](https://blog.csdn.net/qq_27680317/article/details/79970590)
[Hibernate validator使用以及自定义校验器注解](https://blog.csdn.net/dh554112075/article/details/80790464)