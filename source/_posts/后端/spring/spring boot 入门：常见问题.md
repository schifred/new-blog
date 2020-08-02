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

#### 启动时报端口号被占用

查看并杀死占用端口号的进程

```bash
# windows：
netstat -ano |findstr 18080
taskkill /f /t /im 23384
# mac 
lsof -i :18080
kill -9 23384
```