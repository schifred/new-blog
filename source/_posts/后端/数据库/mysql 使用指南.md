---
title: mysql 使用指南
category:
  - 数据库
tags:
  - 数据库
  - mysql
keywords: 'sql,mysql'
abbrlink: 9b6215a9
date: 2019-11-18 00:00:00
updated: 2020-07-19 00:00:00
---

### mac 安装使用示例

```bash
brew install mysql # mac 下使用 Homebrew 安装 mysql https://www.jianshu.com/p/d3f7e7402449

mysql.server start # 启动
mysql.server stop
mysql.server restart

mysql -u root -p # 根用户连接数据库

SHOW DATABASES; # 列出数据库名
use DATABASE_NAME; # 切换默认数据库
SHOW TABLES; # 列出表名
SHOW COLUMNS FROM TABLE_NAME; # 列出某表中的所有属性
```

#### QA

1. 启动时报错 The server quit without updating PID file：https://javawind.net/p141
2. 查看运行端口：https://blog.csdn.net/qq_40604853/article/details/81334477

### 客户端访问软件

navicat，mysql workbench

### 数据类型

* TINYINT：1 字节整数；SMALLINT：2 字节整数；MEDIUMINT：3 字节整数；INT、INTEGER：4 字节整数；BIGINT：8 字节整数
* FLOAT：4 字节单精度浮点数；FLOAT：8 字节双精度浮点数；DECIMAL(M,D)：小数
* DATE：年月日日期；TIME：时分秒时间；YEAR：年份；DATETIME：年月日时分秒；TIMESTAMP：年月日时分秒 + 时间戳
* CHAR：定长字符串，0-255 字节
* VARCHAR：变长字符串，0-65535 字节
* TINYBLOB：不超过 255 个字符的二进制字符串，0-255 字节
* TINYTEXT：短文本字符串，0-255 字节
* BLOB：二进制形式的长文本数据，0-65535 字节
* TEXT：长文本数据，0-65535 字节
* MEDIUMBLOB：二进制形式的中等长度文本数据，0-16777215 字节
* MEDIUMTEXT：中等长度文本数据，0-16777215 字节
* TLONGBLOB：二进制形式的极大文本数据，0-4294967295 字节
* LONGTEXT：极大文本数据，0-4294967295 字节

常用的有 TINYINT 设置布尔值、INT 枚举值、DATE 日期、DATETIME 时间、VARCHAR 字符、TEXT 长文本。

### 命令

#### CREATE、DROP

CREATE、DROP 语句用来创建或删除数据库、建表或删表。

```sql
/** 创建数据库 **/
DROP DATABASE RUNOOB;
CREATE DATABASE RUNOOB;
/** 建表 **/
DROP TABLE `runoob_tbl`;
CREATE TABLE IF NOT EXISTS `runoob_tbl`(
   `runoob_id` INT UNSIGNED AUTO_INCREMENT,
   `runoob_title` VARCHAR(100) NOT NULL,
   `runoob_author` VARCHAR(40) NOT NULL,
   `submission_date` DATETIME NOT NULL DEFAULT NOW(),/** 设置默认值 **/
   PRIMARY KEY ( `runoob_id` )
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

#### SELECT、INSERT、UPDATE、DELETE

WHERE 语句为 sql 语句设置条件。可选的条件操作符包含 =、<>（不等于）、!=、>、<、>=、>=、LIKE 等。LIKE 语句需要与 % 配合使用，% 表示通配符。没有结合 % 的 LIKE 语句与 = 效果等同。

```sql
/** 查 **/
SELECT * from runoob_tbl WHERE runoob_author LIKE '%COM';
/** 增 **/
INSERT INTO runoob_tbl
  (runoob_title, runoob_author, submission_date)
  VALUES
  ("JAVA 教程", "RUNOOB.COM", '2016-05-06');
/** 改 **/
UPDATE runoob_tbl SET runoob_title='学习 C++' WHERE runoob_id=3;
/** 删 **/
DELETE FROM runoob_tbl WHERE runoob_id=3;
```

#### ORDER BY 排序

ORDER BY 语句设定按哪个字段哪种方式来进行排序。默认排序方式 ASC 升序排列，可选 DESC 降序排列。

```sql
# 创建雇员表
DROP TABLE IF EXISTS `employee_tbl`;
CREATE TABLE `employee_tbl` (
  `id` int(11) NOT NULL,
  `name` char(10) NOT NULL DEFAULT '',
  `date` datetime NOT NULL,
  `singin` tinyint(4) NOT NULL DEFAULT '0' COMMENT '登录次数',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

# 从雇员表中检索数据，按日期升序排列
SELECT * from employee_tbl ORDER BY date ASC;
```

#### UNION 求并集

UNION 语句用于连接两个以上的 SELECT 语句的结果组合到一个结果集合中。多个 SELECT 语句会删除重复的数据。UNION ALL 语句可查出重复数据；UNION DISTINCT 语句会剔除重复的数据。UNION 语句默认采用 UNION DISTINCT 查询模式。

```sql
# 从 Websites、apps 汇总查询数据，结果集包含 country、name 字段，name 字段的值可能是 app_name
SELECT country, name FROM Websites
WHERE country='CN'
UNION ALL
SELECT country, app_name FROM apps
WHERE country='CN'
ORDER BY country;
```

#### GROUP BY 分组

GROUP BY 语句根据一个或多个列对结果集进行分组。在分组的列上可以使用 COUNT, SUM, AVG 等函数。

```sql
# 创建雇员表
DROP TABLE IF EXISTS `employee_tbl`;
CREATE TABLE `employee_tbl` (
  `id` int(11) NOT NULL,
  `name` char(10) NOT NULL DEFAULT '',
  `date` datetime NOT NULL,
  `singin` tinyint(4) NOT NULL DEFAULT '0' COMMENT '登录次数',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

# 从雇员表中检索数据，按姓名分组，统计每个人有多少条
SELECT name, COUNT(*) FROM employee_tbl GROUP BY name;

# 从雇员表中检索数据，按姓名分组，统计登录总次数
# WITH ROLLUP 在分组的基础上再进行统计，登录总次数一行的 name 值为 NULL
# coalesce(column_name_a, column_name_b, column_name_c) 如果 column_name_a 为 NULL，以 column_name_b 代替
SELECT coalesce(name, '总数'), SUM(singin) as singin_count FROM employee_tbl GROUP BY name WITH ROLLUP;
```
