---
title: java 知识点
category:
  - 后端
  - java
tags:
  - 后端
  - java
keywords: 'java'
abbrlink: 23d7b8af
date: 2019-12-21 00:00:00
updated: 2019-12-21 00:00:00
---

### 注解

java 注解与 js 装饰器拥有相当不同的实现方式：java 有编译过程，因此可以使用语法约定注解的意义，脱离了编译过程，注解将毫无意义；js 引擎却没有对装饰器给予有效支持，因此 js 装饰器在 babel 等工具中表现为使用高阶函数封装原类或方法。java 注解本质上都是集成了 java.lang.Annotation 的接口，接口名前加 @interface 标识声明。有以下四种元注解可以标识注解的目标、生命周期、是否包含在 JavaDoc 文档中、是否允许子类继承，分别是 @Target、@Retention、@Documented、@Inherited。

@Target 注解包含以下值：

* ElementType.TYPE：允许被修饰的注解作用在类、接口和枚举上
* ElementType.FIELD：允许作用在属性字段上
* ElementType.METHOD：允许作用在方法上
* ElementType.PARAMETER：允许作用在方法参数上
* ElementType.CONSTRUCTOR：允许作用在构造器上
* ElementType.LOCAL_VARIABLE：允许作用在本地局部变量上
* ElementType.ANNOTATION_TYPE：允许作用在注解上
* ElementType.PACKAGE：允许作用在包上

@Retention 注解包含以下值：

* RetentionPolicy.SOURCE：当前注解编译期可见，不会写入 class 文件
* RetentionPolicy.CLASS：类加载阶段丢弃，会写入 class 文件
* RetentionPolicy.RUNTIME：永久保存，可以反射获取

更多内容参考 [JAVA 注解的基本原理](https://www.cnblogs.com/yangming1996/p/9295168.html)。

### 类加载器

java 文件经编译器编译成 class 文件，class 文件由虚拟机加载并创建对应的 java.lang.Class 对象实例。虚拟机提供了三种类加载器：Bootstrap 启动类加载器、Extension 扩展类加载器、System 系统类加载器。启动类加载器主要加载的虚拟机自身所需的类（使用 C++ 实现），包含 <JAVA_HOME>/lib 路径下的核心类库以及 -Xbootclasspath 参数指定路径下的 jar 包。扩展类加载器负责加载”标准的扩展“，包含 <JAVA_HOME>/lib/ext 目录下或者由系统变量 -Djava.ext.dir 指定路径中的类库。系统类加载器负责加载系统类路径 java -classpath 或 -D java.class.path 指定路径下的类库。虚拟机对 class 文件采用按需加载的方式，并使用双亲委派模式，即先有父类加载器处理，如果加载失败，再交由子类加载器处理。类加载器的典型使用如下：

```java
// 1. 创建 URL 资源
URL url = new URL("file:///path/to/plugin.jar");
// 2. 创建 URLClassLoader 实例，URLClassLoader 会通过给定的 URL 加载类
URLClassLoader pluginLoader = new URLClassLoader(new URL[] { url });
// 3. 加载指定包下的类
Class<?> cl = pluginLoader.loadClass("mypackage.MyClass");
```

自定义的加载器可以通过集成 ClassLoader 实现。ClassLoader#loadClass 方法的职责是将类的加载操作委托给父类加载器，当父类加载器无法加载时，就会转而使用 ClassLoader#findClass。因此自定义的加载逻辑推荐实现在 ClassLoader#findClass 方法中。

每个线程都由对类加载器的引用，称为上下文类加载器。主线程的上下文加载器是系统类加载器。当新线程创建时，它的上下文类加载器会被设置成创建该线程的上下文类加载器。线程的上下文类加载器可以通过 thread#setContextClassLoader 方法改写。

更多内容参考 [深入理解Java类加载器](https://www.cnblogs.com/mybatis/p/9396135.html)。

### 反射

java 的反射机制用于动态获取任意一个对象的方法和属性等信息以及动态调用对象方法。更多内容参考 [Java高级特性——反射](https://www.jianshu.com/p/9be58ee20dee)。

### 代理