---
title: RMI
category:
  - 微服务
  - 远程通信
tags:
  - 微服务
  - 远程通信
keywords: '远程通信'
abbrlink: 822cfee5
date: 2019-12-21 00:00:00
updated: 2019-12-21 00:00:00
---

### RMI

分布式编程即当客户端调用某方法时，产生一个请求发送到服务器上，然后取回响应。RMI（远程方法调用）的实现思路植根于：在客户端和服务端均安装代理，由客户端代理将客户端方法调用转换成请求（包含服务器地址、所调用方法、传参等信息），服务端代理通过请求执行服务端方法，并向客户端发送响应。服务端接口实例（实现 Remote 接口、扩展 UnicastRemoteObject 类）须通过 InitialContext 实例注册到 RMI 注册表中，客户端就可以通过 InitialContext#list 或 InitialContext#lookup 方法获取到代理对象（称为存根）。

### RPC

### web 服务

### 参考

[RMI与RPC的区别](https://cloud.tencent.com/developer/article/1353191)
[Java RMI 简介及其优劣势总结](https://blog.csdn.net/mingtianhaiyouwo/article/details/50513577)
[EJB到底是什么？](https://blog.csdn.net/kouzhaokui/article/details/89176541)
[什么是RPC？](https://www.jianshu.com/p/7d6853140e13)