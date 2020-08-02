---
title: spring boot 入门：附录（二） IDEA
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: b600fbee
date: 2019-03-17 12:30:00
updated: 2019-03-17 12:30:00
---

### 快捷键

* 双击 shift 当前项目的所有文件夹中查找内容
* ctrl + f 当前文件查找特定内容
* ctrl + shift + f 查找特定内容
* ctrl + n 查找类
* ctrl + shift + n 查找文件
* ctrl + r 当前文件替换内容
* ctrl + shift + r 当前项目替换内容
* shift + F6 重命名类等
* alt + insert 类中使用自动生成方法，项目中使用创建类
* alt + enter 消除黄色警告，自动优化代码
* ctrl + alt + l 代码格式化
* ctrl + alt + o 移除无效导入

更多内容可戳 [IDEA 快捷键](https://blog.csdn.net/xkzju2010/article/details/69487723)。

### 常见问题

#### 项目目录结构折叠问题

点击 Project 一行上齿轮图标，取消 Hide Empty Middle Packages 选中态

#### 指定 profile 方式启动

VM OPTIONS 添加 -Dspring.profiles.active=local

#### Java 1.5 版本过低

检查 Project Structure - Project | Modules 指定的当前项目 jdk 版本、以及 Setting - Java Compiler 指定的 Java 版本是否正确。