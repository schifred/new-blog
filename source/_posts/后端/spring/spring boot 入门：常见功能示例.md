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

### 图片校验

我们可使用 google 开源的 kaptcha，首先添加 com.github.penggle/kaptcha 依赖。其次添加 Kaptcha 配置类，

```java
@Configuration
public class KaptchaConfig {
    @Bean
    public DefaultKaptcha getDDefaultKaptcha() {
        DefaultKaptcha dk = new DefaultKaptcha();
        Properties properties = new Properties();

        // 图片边框
        properties.setProperty("kaptcha.border", "yes");
        // 边框颜色
        properties.setProperty("kaptcha.border.color", "105,179,90");
        // 字体颜色
        properties.setProperty("kaptcha.textproducer.font.color", "red");
        // 图片宽
        properties.setProperty("kaptcha.image.width", "110");
        // 图片高
        properties.setProperty("kaptcha.image.height", "40");
        // 字体大小
        properties.setProperty("kaptcha.textproducer.font.size", "30");
        // session key
        properties.setProperty("kaptcha.session.key", "code");
        // 验证码长度
        properties.setProperty("kaptcha.textproducer.char.length", "4");
        // 字体
        properties.setProperty("kaptcha.textproducer.font.names", "宋体,楷体,微软雅黑");
        Config config = new Config(properties);
        dk.setConfig(config);

        return dk;
    }
}

@Controller
@Slf4j
public class VerifyCodeController extends BaseController {
    @Autowired
    DefaultKaptcha kaptcha;

    @GetMapping("/code")
    public void refreshVerifyCode(HttpServletRequest req, HttpServletResponse res) throws IOException {
        res.setHeader("Cache-Control", "no-store, no-cache");
        res.setContentType("image/jpeg");
        String capText = kaptcha.createText();
        HttpSession session = req.getSession();
        session.setAttribute("KAPTCHA_SESSION_KEY", capText);
        session.setAttribute("KAPTCHA_SESSION_DATE", Long.valueOf(System.currentTimeMillis()));
        BufferedImage bi = kaptcha.createImage(capText);
        ServletOutputStream out = res.getOutputStream();
        log.info("session 创建成功，id:{}；图片校验码:{}", session.getId(),capText);
        ImageIO.write(bi, "jpg", (OutputStream) out);
    }

    private boolean checkVerifyCode(String verifyCode, HttpServletRequest req){
        HttpSession session = req.getSession(false);
        if (session == null){
            log.info("session 已销毁，验证码校验无效");
            return false;
        }

        String kaptcheExpected = (String)session.getAttribute("KAPTCHA_SESSION_KEY");
        String kaptcheCreateTime = (String)session.getAttribute("KAPTCHA_SESSION_DATE");
        log.info("session id:{}；图片校验码期望值:{}；提交值:{}", session.getId(), kaptcheExpected, verifyCode);
        if (verifyCode == null || kaptcheExpected == null || !verifyCode.equalsIgnoreCase(kaptcheExpected))
            return false;
        session.removeAttribute("KAPTCHA_SESSION_KEY");
        session.removeAttribute("KAPTCHA_SESSION_DATE");
        return true;
    }
}
```

### content-type

* [content-type 对照表](https://tool.oschina.net/commons/)
* [office文件的 content-type](https://www.jianshu.com/p/4b09c260f9b2)