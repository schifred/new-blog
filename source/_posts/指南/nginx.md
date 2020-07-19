---
title: nginx
category:
  - 指南
tags:
  - 指南
  - 部署
keywords: 'nginx,部署'
abbrlink: a30037a7
date: 2019-02-16 00:00:00
updated: 2019-02-16 00:00:00
---

### 安装并启动

1. nginx主站 下载 nginx
2. cd 到 nginx 目录，start nginx 启动
3. 修改 conf/nginx.conf 全局配置，nginx -s reload 重启

### 命令

```bash
start nginx 启动
nginx -s stop 强制退出
nginx -s quit 平滑退出
nginx -s reload 修改配置后，平滑重启

taskkill /F /IM nginx.exe > null 强制杀死 nginx 进程
taskkill /fi “imagename nginx.exe” 查看 nginx 进程
```

### 配置

```yml
worker_process 1;

events {
  worker_connection 1024;
}

http {
  include mime.types; # 缺失会导致 css 解析无效
  default_type application/octet-stream;

  gzip on;

  server {
    listen 80;
    server_name localhost;
    root D:/git/test;
  }
}
```

### 参考

[Nginx安装及配置详解](https://www.cnblogs.com/fengff/p/8892590.html)
[nginx常用配置—-作为web服务端](https://blog.csdn.net/wangzhenyu177/article/details/78679053)
[nginx for Windows](http://nginx.org/en/docs/windows.html)