---
title: spring boot 入门：常见功能示例
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 9999f3ca
date: 2019-03-17 06:30:00
updated: 2019-03-17 06:30:00
---

### cors 跨域

实现 cors 跨域有多种方式，controller 方法中设置消息头、过滤器中设置消息头或配置类中设置跨域准入信息。这里只贴示第三种方式：

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**").
      .allowedOrigins("*")// 允许访问的域名
      .allowedMethods("*")// 允许任何方法（post、get等）
      .allowedHeaders("*")// 允许任何请求头
      .allowCredentials(true)// 带上cookie信息
      .exposedHeaders(HttpHeaders.SET_COOKIE).maxAge(3600L);// 多少时间内不必发送预检验请求
  }
}
```

### 文件上传

文件上传可借助 MultipartFile。

```java
@PostMapping("/uploadFile")
public Result<FileDTO> uploadFile(@RequestParam("file") MultipartFile file){
  FileDTO fileDTO = storageService.store(file);// 实现存储 service

  return Result.createSuccessResult(fileDTO);
}
```

更多内容可戳 [Uploading Files 文档](https://spring.io/guides/gs/uploading-files/)。

### content-type

* [content-type 对照表](https://tool.oschina.net/commons/)
* [office文件的 content-type](https://www.jianshu.com/p/4b09c260f9b2)