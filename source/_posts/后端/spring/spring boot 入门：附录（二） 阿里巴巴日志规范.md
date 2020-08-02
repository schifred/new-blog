---
title: spring boot 入门：附录（二） 阿里巴巴日志规范
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 45a8ec69
date: 2019-03-17 16:32:00
updated: 2019-03-17 16:32:00
---

这里只截取笔者认为重要、或有影响、或未理解的内容，详情可戳 [这里](https://github.com/alibaba/p3c/tree/master/p3c-gitbook/%E5%BC%82%E5%B8%B8%E6%97%A5%E5%BF%97)。

### 异常处理

* 【参考】（分层异常处理规约）在DAO层，产生的异常类型有很多，无法用细粒度的异常进行catch，使用catch(Exception e)方式，并throw new DAOException(e)，不需要打印日志，因为日志在Manager/Service层一定需要捕获并打印到日志文件中去，如果同台服务器再打日志，浪费性能和存储。在Service层出现异常时，必须记录出错日志到磁盘，尽可能带上参数信息，相当于保护案发现场。如果Manager层与Service同机部署，日志方式与DAO层处理一致，如果是单独部署，则采用与Service一致的处理方式。Web层绝不应该继续往上抛异常，因为已经处于顶层，如果意识到这个异常将导致页面无法正常渲染，那么就应该跳转到友好错误页面，加上用户容易理解的错误提示信息。开放接口层要将异常处理成错误码和错误信息方式返回。
* 【强制】Java 类库中定义的可以通过预检查方式规避的RuntimeException异常不应该通过catch 的方式来处理，比如：NullPointerException，IndexOutOfBoundsException等可以通过判断非null等方式处理。
* 【强制】有try块放到了事务代码中，catch异常后，如果需要回滚事务，一定要注意手动回滚事务。
* 【强制】finally块必须对资源对象、流对象进行关闭，有异常也要做try-catch。说明：如果JDK7及以上，可以使用try-with-resources方式。
* 【推荐】防止NPE，是程序员的基本修养，注意NPE产生的场景：
1）返回类型为基本数据类型，return包装数据类型的对象时，自动拆箱有可能产生NPE。
反例：public int f() { return Integer对象}， 如果为null，自动解箱抛NPE。
2） 数据库的查询结果可能为null。
3） 集合里的元素即使isNotEmpty，取出的数据元素也可能为null。
4） 远程调用返回对象时，一律要求进行空指针判断，防止NPE。
5） 对于Session中获取的数据，建议NPE检查，避免空指针。
6） 级联调用obj.getA().getB().getC()；一连串调用，易产生NPE。
正例：使用JDK8的Optional类来防止NPE问题。
* 【推荐】定义时区分unchecked / checked 异常，避免直接抛出new RuntimeException()，更不允许抛出Exception或者Throwable，应使用有业务含义的自定义异常。推荐业界已定义过的自定义异常，如：DAOException / ServiceException等。
* 【参考】对于公司外的http/api开放接口必须使用“错误码”；而应用内部推荐异常抛出；跨应用间RPC调用优先考虑使用Result方式，封装isSuccess()方法、“错误码”、“错误简短信息”。
说明：关于RPC方法返回方式使用Result方式的理由：
1）使用抛异常返回方式，调用方如果没有捕获到就会产生运行时错误。 2）如果不加栈信息，只是new自定义异常，加入自己的理解的error message，对于调用端解决问题的帮助不会太多。如果加了栈信息，在频繁调用出错的情况下，数据序列化和传输的性能损耗也是问题。

### 日志

* 【强制】应用中不可直接使用日志系统（Log4j、Logback）中的API，而应依赖使用日志框架SLF4J中的API，使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一。 
* 【强制】日志文件推荐至少保存15天，因为有些异常具备以“周”为频次发生的特点。
* 【强制】应用中的扩展日志（如打点、临时监控、访问日志等）命名方式：appName_logType_logName.log。logType:日志类型，推荐分类有stats/monitor/visit等；logName:日志描述。这种命名的好处：通过文件名就可知道日志文件属于什么应用，什么类型，什么目的，也有利于归类查找。
正例：mppserver应用中单独监控时区转换异常，如：
mppserver_monitor_timeZoneConvert.log
说明：推荐对日志进行分类，如将错误日志和业务日志分开存放，便于开发人员查看，也便于通过日志对系统进行及时监控。
* 【强制】对trace/debug/info级别的日志输出，必须使用条件输出形式或者使用占位符的方式。
说明：logger.debug("Processing trade with id: " + id + " and symbol: " + symbol); 如果日志级别是warn，上述日志不会打印，但是会执行字符串拼接操作，如果symbol是对象，会执行toString()方法，浪费了系统资源，执行了上述操作，最终日志却没有打印。
正例：（条件） 
```java
if (logger.isDebugEnabled()) {    
  logger.debug("Processing trade with id: " + id + " and symbol: " + symbol);   
}  
```
正例：（占位符）
```java
logger.debug("Processing trade with id: {} and symbol : {} ", id, symbol);
```
* 【强制】避免重复打印日志，浪费磁盘空间，务必在log4j.xml中设置additivity=false。
* 【推荐】谨慎地记录日志。生产环境禁止输出debug日志；有选择地输出info日志；如果使用warn来记录刚上线时的业务行为信息，一定要注意日志输出量的问题，避免把服务器磁盘撑爆，并记得及时删除这些观察日志。
* 【推荐】可以使用warn日志级别来记录用户输入参数错误的情况，避免用户投诉时，无所适从。如非必要，请不要在此场景打出error级别，避免频繁报警。说明：注意日志输出的级别，error级别只记录系统逻辑出错、异常或者重要的错误信息。


