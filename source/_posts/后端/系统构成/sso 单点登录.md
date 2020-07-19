---
title: sso 单点登录
category:
  - 后端
  - 系统构成
tags:
  - 后端
  - 系统构成
  - sso
keywords: 'sso,spring'
abbrlink: 8f345e41
date: 2019-12-01 00:00:00
updated: 2019-12-01 00:00:00
---

单应用，客户端传入的用户信息会使用 session 记录，然后将 sessionId 通过 cookie 回传给客户端，以便在下次发起请求时获取 session 存储的用户信息。

多子域应用，如 app1.a.com、app2.a.com，在用户访问 sso.a.com 并作登录操作时，服务器端可以将 cookie 的 domain 设为 taobao.com，从而使不同子域发起请求时能携带 sessionId，然后可使用 spring-session 等技术实现跨子域之间的共享 session，从而实现多子域的单点登录。

跨域应用，用户访问 a 系统，将重定向到 sso 系统进行登录，sso 系统将用户登录信息存入 session；sso 系统将访问地址重定向到 a 系统并传递 service ticket，a 系统持有 service ticket 从 sso 系统中获取用户信息并将写入自身的 session 中，同时传递 cookie 到客户端。若用户已通过 b 系统完成 sso 登录，a 系统可以在 session 失效时重走 sso 登录流程，以取得用户信息。

除了存储用户信息，sso 系统也会将调用用户信息的子系统也缓存起来，以便在注销时通知子系统删除各自的局部回话，以实现全局注销。

[sso简单原理及实现](https://www.cnblogs.com/zh94/p/8352943.html) 贴出了这张时序图：

![image](sso1.png)

对于 sso-client、sso-server 实现，[sso简单原理及实现](https://www.cnblogs.com/zh94/p/8352943.html) 给出了简单的示例 [simple-sso](https://github.com/sheefee/simple-sso)。

### sso-client

对于登录，sso-client 会拦截请求，若用户已登录，则放行；未登录，且请求参数中带有 sso 系统传入的 service ticket，从 sso 系统获取用户登录信息；其余未登录且须登录的情况，重定向到 sso 系统。

对于登出，sso-client 会调用 sso 系统发送注销请求，再由 sso 系统通知其他子系统销毁局部回话。

登陆流程源码见于 [SsoFilter.java](https://github.com/a466350665/smart/blob/master/smart-sso/smart-sso-client/src/main/java/com/smart/sso/client/SsoFilter.java)，主要有两个操作：

1. 重定向到 sso-server 登录页，促使用户完成登录操作。
2. 从请求参数中获取 sso-server 重定向带回的 service ticket，使用 service ticket 从 sso-server 获取用户信息。
3. 客户登录状态下进行，直接使用客户端发送的 sessionId，并通过 sso-server 校验 sessionId 的有效性。

### sso-server

对于登录，sso-server 会重定向到登录页面，在用户发起登录请求、服务端缓存用户信息后，sso 系统带着 service ticket 重定向到 sso-client，以使其获得登录信息。

对于登出，sso-server 会销毁缓存的用户信息以及子系统记录，并通知其余子系统销毁局部回话。

登陆流程源码见于 [LoginController.java](https://github.com/a466350665/smart/blob/master/smart-sso/smart-sso-server/src/main/java/com/smart/sso/server/controller/LoginController.java)，主要有两个操作：

1. 跳转到登录页，如果 cookie 带有 sessionId，重定向到 sso-client，访问路径上携带 service ticket。
2. 用户尝试登陆，重新生成带有 sessionId 的 cookie，并重定向到 sso-client，访问路径上携带 service ticket。
3. service ticket 自有逻辑，以定时器方式设置 30 分钟后过期。

登出流程源码见于 [LogoutController.java](https://github.com/a466350665/smart/blob/master/smart-sso/smart-sso-server/src/main/java/com/smart/sso/server/controller/LogoutController.java)，其执行逻辑为：使 service ticket 失效。登出流程可以通过 sso-client 发起。

### 参考

[sso简单原理及实现](https://www.cnblogs.com/zh94/p/8352943.html) 
[单点登录（SSO）看这一篇就够了](https://developer.aliyun.com/article/636281)
[sso 单点登录平台](https://github.com/kawhii/sso)
[cas 单点登录](https://blog.csdn.net/u010475041/category_7156505.html)
[Enterprise Single Sign-On for All](https://apereo.github.io/cas/5.1.x/index.html)
[smart 单点登录应用](https://github.com/a466350665/smart)