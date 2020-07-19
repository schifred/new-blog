---
title: spring boot（七）：定时任务
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 2290be3f
date: 2019-03-17 05:00:00
updated: 2019-03-17 05:00:00
---

启动类中添加 @EnableScheduling 注解开启定时任务。

```java
@ComponentScan(basePackages = "com.example.demo.*")
@SpringBootApplication
@EnableScheduling
public class DemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

实现定时任务 service。

```java
@Component
public class QuartzService {
    // 每分钟执行
    @Scheduled(cron = "0 0/1 * * * ?")
    public void timerToNow(){
        System.out.println("now time:" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
    }
}
```

### 参考

[cron表达式](https://www.cnblogs.com/tommyli/p/3760671.html)
