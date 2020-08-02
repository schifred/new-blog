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
date: 2019-03-17 07:00:00
updated: 2019-03-17 07:00:00
---

Q：启动时报端口号被占用
A：查看并杀死占用端口号的进程

```bash
# windows：
netstat -ano |findstr 18080
taskkill /f /t /im 23384
# mac 
lsof -i :18080
kill -9 23384
```

Q：SpringBoot 启动失败 Failed to determine a suitable driver class 问题解决方案
A：使用数据驱动的情况，配置文件中没有 dataSource 配置信息，可能是启动配置没有设置或者错误设置  -Dspring.profiles.active=${env}

Q：启动 missing ServletWebServerFactory bean.报错
A：https://www.jb51.cc/spring/432649.html

Q：class path resource [logback-config.xml] cannot be resolved to URL because it does not exist
A：https://www.cnblogs.com/uqing/p/9812547.html

Q：Ambiguous mapping. Cannot map '*Controller' method
A：RequestMapping 冲突 https://blog.csdn.net/m0_37961948/article/details/82791129

Q：error Type referred to is not an annotation type 报错
A：注解引用路径不对

Q：java.lang.IllegalArgumentException: Could not resolve placeholder 报错
A：bootstrap.yaml 配置未顶格

Q：Unrecognized SSL message, plaintext connection?
A：将 http 请求错写成 https

Q：redis: WRONGTYPE Operation against a key holding the wrong kind of value
A：同样的键已有数据存入，第二次存入的数据类型与第一次不同 https://blog.csdn.net/hanchao5272/article/details/79051364