---
title: spring boot 入门：接入 swagger2
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: d9e408c0
date: 2019-03-17 05:55:00
updated: 2019-03-17 05:55:00
---

接入 swagger2 可用于为工程提供在线 api 文档服务。按以下流程可完成接入：

1. 添加 springfox-swagger2、springfox-swagger-ui 依赖。
2. 使用 @Configuration、@EnableSwagger2 注解编写 Swagger 配置类。
3. 在 Controller、实体类中使用 Swagger 注解 @Api 等。
4. 访问 /swagger-ui.html 查看接口文档（或者 /v2/api-docs 查看全量 json 数据）。

通常配置类模板如下，即会对模块中的所有 controller 进行扫描并据此生成文档。

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket customDocket() {
        return new Docket(DocumentationType.SWAGGER_2)// 使用 swagger2 规范
                .apiInfo(getApiInfo())// 作为接口文档的基本信息
                .select()// 返回 ApiSelectorBuilder 实例
                .apis(RequestHandlerSelectors.any())// 匹配包的形式指定须生成文档的 controller
                .paths(PathSelectors.any())// 指定须生成文档的接口路径
                .build();
    }
    private ApiInfo getApiInfo() {
        Contact contact = new Contact("团队名", "www.my.com", "my@my.com");
        return new ApiInfoBuilder()
                .title("文档标题")
                .description("文档描述")
                .contact(contact)// 联系方式
                .version("1.1.0")// 版本
                .build();
    }
}
```

实体类可用注解：
* @ApiModel(description="...")：描述实体类。
* @ApiModelProperty：描述属性，描述信息包含 notes、example、position 等。

控制器可用注解：
* @Api(tags="接口所在的类")：描述实体类。
* @ApiOperation：描述方法，描述信息包含 value、notes、httpMethod 等。
* @ApiImplicitParams：包含一组 @ApiImplicitParam 描述参数。
* @ApiImplicitParam：描述参数，描述信息包含 name、value、required、paramType、dataType、defaultValue 等。
* @ApiResponse：描述响应，描述信息包含 code、message 等。

```java
@RestController
@Api(tags="接口总描述")
public class HelloController {
    @GetMapping(value = "/hello")
    @ApiOperation(value = "接口名", notes = "接口描述", httpMethod = "GET")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "length",value = "参数1", required = true, paramType = "path"),
            // @ApiImplicitParam(name = "size",value = "参数2", required = true, paramType = "query"),
            // @ApiImplicitParam(name = "page",value = "参数3", required = true, paramType = "header"),
            // @ApiImplicitParam(name = "total",value = "参数4", required = true, paramType = "form"),
            // @ApiImplicitParam(name = "start",value = "参数5",dataType = "string", paramType = "body")
    })
    public String getSomething(){
        // ...
    }
}
```

### JSR-303

借助 springbean-fox-validators，生成的 api 文档将包含实体类上的 JSR-303 注解 @NotNull 等（这些注解又见用于 hibernate-validator）。

同时，Swagger 配置类需要添加 @Import(BeanValidatorPluginsConfiguration.class) 注解。