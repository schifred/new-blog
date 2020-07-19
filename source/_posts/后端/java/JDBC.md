---
title: JDBC
category:
  - 后端
  - java
tags:
  - 后端
  - java
keywords: 'JDBC,java'
abbrlink: c32356be
date: 2019-11-21 00:00:00
updated: 2019-11-21 00:00:00
---

Java 数据库连接（JDBC）API 用于对接不同的数据库 —— 由遵守不同网络协议的数据库厂商提供第三方驱动程序，再由 Java 提供一个驱动管理器和一套 API，驱动程序就会注册到驱动管理器中，在此基础上，调用 API 即能访问驱动管理器，最后通过驱动程序与实际的数据库通信。驱动程序的实现经历过多次演进，目前以纯 Java 语言实现，它能将 JDBC 请求直接翻译成数据库相关的协议。

![image](jdbc.jpeg)

使用 JDBC 的前置操作包含：获悉数据库 url 地址如 jdbc:mysql://127.0.0.1:3306/test?characterEncoding=UTF-8；下载驱动程序 jar 文件并注册到驱动管理器上（部分驱动不需要注册）；下载并启动特定数据库。完成前置操作后，就可以连接数据库并执行 sql 了。典型的 JDBC 调用操作如下：

```java
// 用户名或密码可以放在配置文件中或远程管理中心中
String url = "jdbc:mysql://127.0.0.1:3306/test?characterEncoding=UTF-8";
String username = "admin";
String password = "123456";
Connection conn = DriverManager.getConnection(url, username, password);

Statement stat = conn.createStatement();
stat.executeUpdate("CREATE TABLE Greeting (Message CHAR(20))");
```

JDBC 接口设计包含如下内容：

1. 以 Connection 抽象数据库连接，通过 DriverManager.getConnection 创建。连接通过 connection.close 关闭，避免长连接占用有限的数据库资源（数据库有最大并发连接数、最大并发语句数）（与此同时，频繁建立和关闭数据库连接也是高昂的消耗，因此 web 容器或应用服务器开发商会提供连接池机制。连接池保持与数据库的长连接，不存在物理关闭，sql 操作可以重用连接池）。
2. 以 Statement 抽象针对数据库的表操作，通过 connection.createStatement 创建，同样能通过调用 close 方法关闭。Statement 对象实现 executeQuery, executeUpdate, execute 方法用于增删改查操作。
3. 以 ResultSet 抽象查询结果集，并缓存在 Statement 对象中，可通过 statement.getResultSet 获取（每个结果集只能获取一次）。部分结果集允许单条 Select 语句查出多个结果集，通过 statement.getMoreResults 使 getResultSet 指向下一条数据。
4. ResultSet 可以便捷地访问列数据，主要抽象操作包含 getXxx(int columnNumber), getXxx(String columnName), getObject(int columnNumber, Classtype), getObject(String columnLabel, Classtype) 等。Xxx 为 String, Int, Double, Date, Blob, Clob。
5. Statement 再进一步，以 PreparedStatement 抽象预编译的表操作，通过 connection.PrepareStatement 创建。它能复用基本的 sql 语句（此时为预备语句）并对编译结果作缓存，? 部分语句可在特定场合下填充为实际值（如 where name = ? 查询特定人员）。preparedStatement.setXxx, preparedStatement.clearParameters 可用于设置或清除参数（即实际值）。
6. 支持通过 ResultSet 滚动查询、更新操作。
7. 以 RowSet 缓存查询结果。
8. 支持查询元数据。
9. 支持事务操作。6、7、8、9 均见下文。

JDBC 另外解决的问题包含：（日期格式等出入库时）转义；获取自动生成的主键。

### 可滚动、可更新

部分数据库的驱动程序支持结果集可滚动、可更新，即直接通过 ResultSet 翻查上一条或下一条数据，更新 ResultSet 数据内容即可更新表数据。在执行结果集滚动、更新操作前，首先需要通过 DatabaseMetaData 接口的 supportsResultSetType, supportsResultSetConcurrency 方法进行判断。

### 行集

可滚动、可更新操作需要与数据库保持连接，消耗数据库资源，因此 JDBC 又推出了扩展自 ResultSet 的行集 RowSet 接口。RowSet 接口有以下子接口：CachedRowSet 缓存行集、WebRowSet 缓存行集（保存为 XML）、FilteredRowSet / JoinRowSet 可操作行集（select、join 操作）、JdbcRowSet 继承 get、set 方法的 “bean”。RowSet 允许将 ResultSet 数据填充到行集中；直接执行 sql 语句；设置分页；重新与数据库建立连接，并将数据回写到数据库中（回写时会校验原始值是否与数据库中存储值相同，保证一致性；回写复杂结果一般是不允许的）。RowSet 的典型操作如下：

```java
// 或 CachedRowSet crs = new com.sun.rowset.CachedRowSetImpl();
RowSetFactory factory = RowSetProvider.newFactory();
CachedRowSet crs = factory.createCachedRowSet();
crs.setCommand("SELECT * FROM Books WHERE PUBLISHER = ?");
crs.setString(1, publisherName);
crs.execute();
```

### 元数据

JDBC 可用于获取元数据：数据库元数据信息、结果集的元数据信息以及预备语句的元数据信息。数据库元数据信息包含表及字段信息、数据库支持的功能。典型操作如下：

```java
Connection conn = DriverManager.getConnection(url, username, password);
DatabaseMetaData meta = conn.getMetaData();
ResultSet mrs = meta.getTables(null, null, null, new String[]{ "Table" });// 获取所有表信息
```

### 事务

一组语句构成一个事务。在一组语句执行过程中，如果中间某个语句执行失败，那么就需要对之前的所有操作进行回滚 rollback。数据库连接默认处于自动提交模式，即 sql 语句一旦被执行就会被提交 commit，无法对其回滚。因此在使用之前，首先需要关闭自动提交模式 setAutoCommit(false)。部分数据库支持更为细致的回滚操作：通过在一组语句执行过程中设置保存点 setSavepoint，就允许后续的回滚操作撤回到保存点位置。同时 JDBC 允许对数据库进行批量更新操作，通过 conn.addBatch, conn.executeBatch 执行。

### 参考

《Java 核心技术卷二》
[JDBC Developer’s Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/jjdbc/toc.htm)