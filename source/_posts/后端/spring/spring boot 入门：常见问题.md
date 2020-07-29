---
title: spring boot 入门：常见问题
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 9999f3ca
date: 2020-04-30 06:00:00
updated: 2020-04-30 06:00:00
---

### 文件上传

文件上传可借助 MultipartFile。

```java
@PostMapping("/uploadFile")
public Result<FileDTO> uploadFile(@RequestParam("file") MultipartFile file){
  FileDTO fileDTO;
  try {
    fileDTO = storageService.store(file);// 实现存储 service
  } catch (DAOException de) {
    return Result.createFailResult(ResponseConstants.DAO_EXCEPTION);
  } catch (BusinessException be) {
      return Result.createFailResult(String.valueOf(
        ResponseConstants.BUSINESS_EXCEPTION.getCode()), be.getMessage()
      );
  } catch (Exception e) {
    return Result.createFailResult(ResponseCodeConstants.UNKOWN_SYSTEM_ERROR);
  };

  return Result.createSuccessResult(fileDTO);
}
```

spring boot 官网[Uploading Files 文档](https://spring.io/guides/gs/uploading-files/)提供了上传文件、下载文件、列出文件清单的示例。

### content-type

* [content-type 对照表](https://tool.oschina.net/commons/)
* [office文件的 content-type](https://www.jianshu.com/p/4b09c260f9b2)