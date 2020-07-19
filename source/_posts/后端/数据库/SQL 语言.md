---
title: SQL 语言
category:
  - 数据库
tags:
  - 数据库
  - sql
keywords: '数据库,sql'
abbrlink: b8645cb3
date: 2020-05-20 00:00:00
updated: 2020-05-20 00:00:00
---

SQL 关系数据库语言标准包含两个部分：

* DDL —— Data Definition Language 数据定义语言：用于定义数据库结构和数据的访问控制。
* DML —— Data Mnipulation Language 数据操作语言：用于检索和更新数据。

《数据库系统——设计、实现与管理（基础篇）》选用 BNF —— Backus Naur Form 巴克斯范式定义 SQL 语句：

* 大写字母：表示保留字，须准确拼写。
* 小写字母：用户自定义字。
* | 竖线：表示从选项中进行选择。
* 大括号：表示所需元素。
* 中括号：表示可选元素。
* 省略号：表示某一项可以重复零到多次。

### DDL

#### 数据类型

SQL 标识符用于指定表名、视图名和列，标识符可使用大小写字母、数字、下划线书写，一般不能大于 128 个字符，不能有空格，必须以字母开头。

SQL 数据类型包含：

* BOOLEAN：布尔型。
* CHAR、VARCHAR：字符型，设定格式如 VARCHAR[(length)]。
* BIT、BIT VARYING：位类型，设定格式如 BIT [VARYING] [(length)]。
* NUMERIC、DECIMAL、INTEGER、SMALLINT、BINGINT：定点数型，设定格式如 NUMERIC[(precision [, scale])]、INT。
* FLOAT、REAL、DOUBLE PRECISION：浮点数型，设定格式如 FLOAT[(precision)]、REAL。
* DATE、TIME、TIMESTAMP：日期时间型，设定格式如 DATE、TIME[timePrecision] [WITH TIME ZONE]。
* INTERVAL：间隔型，设定格式如 INTERVAL YEAR(2) TO MONTH。
* CHARACTER LARGE OBJECT、BINART LARGE OBJECT：大对象型，包含 BLOB 二进制大对象、CLOB 字符大对象、NCLOB 自然字符大对象。

#### 标量运算符

标量运算符除 +、-、*、/ 算术运算符外，还有：

* OCTET_LENGTH：返回字符串的长度（位长度除以 2）。
* CHAR_LENGTH：返回字符串的长度。
* CAST：转换数据类型，如 CAST (5.2E6 AS INTEGER)。
* ||：连接连个字符串或位字符串。
* LOWER、UPPER：大小写转换。
* TRIM：裁剪字符串。
* POSITION(a IN b)：返回子字符串 a 在 b 中的位置。
* SUBSTRING(a FROM m TO n)：返回子字符串。
* CASE columnName WHEN ‘a’ THEN a ELSE 0 END：条件返回。
* CURRENT_DATE：返回用户当地当前日期。
* CURRENT_TIME：返回当前时间。
* CURRENT_TIMESTAMP：返回当前日期和时间。
* ECTRACT：提取日期和时间间隔中的年、月或日信息。
* ABS：计算绝对值。
* MOD：取模。
* LN：计算自然对数。
* EXP：计算直属函数。
* POWER：幂计算。
* SQRT：计算平方根。
* FLOOR：向下取整。
* CEIL：向上。

### 完整性约束

* 空值约束：NOT NULL、NULL 用于限定列数据是否允许为空值，如 position VARCHAR(10) NOT NULL。
* 域约束：域约束，用于限定枚举值的可能性。

```sql
--- 方式1，使用 CHECK
sex CHAR NOT NULL CHECK (sex IN ('M', 'F'));

--- 方式2，使用 CREATE DOMAIN
CREATE DOMAIN sexType AS CHAR
  DEFAULT 'M'
  CHECK (VALUE IN ('M', 'F'));
sex sexType NOT NULL;

--- 销毁，当域约束 sexType 被表定义所使用时，RESTRICT 会阻止销毁，CASCADE 会级联销毁
DROP DOMAIN sexType [RESTRICT | CASCADE]
```

#### 实体完整性

主关键字可以用 PRIMARY KEY 设定，一个表只能设置一个关键字，但是关键字可以通过组合多个列字段构成，如 PRIMARY KEY(clientNo, propertyNo)。

关键字 UNIQUE 可保证列的唯一性（该列须被声明为 NOT NULL），每个表可以指定多个 UNIQUE 子句。

### 数据定义

#### 创建、删除数据库

```sql
CREATE SCHEMA databaseName
DROP SCHEMA databaseName [RESTRICT | CASCADE]
```

#### 创建、删除表，修改表定义

```sql
CREATE TABLE tableName
  {(columnName dataType [NOT NULL [UNIQUE])
    [DEFAULT defaultOption] [CHECK (searchCondition)] [,...]}
  [PRIMARY KEY (listColumns),]
  {[UNIQUE (listColumns),] [,...]}

DROP TABLE tableName [RESTRICT | CASCADE]

ALTER TABLE tableName
  [ADD [COLUMN] columnName dataType [NOT NULL] [UNIQUE]]
  [DEFAULT defaultOption] [CHECK (searchCOndition)]
  [DROP [COLUMN] columnName [RESTRICT | CASCADE]]
  [ALTER [COLUMN] SET DEFAULT defaultOption]
  [ALTER [COLUMN] DROP DEFAULT]
```

#### 创建、删除索引

索引可以提升数据检索的效率，索引只能在基表上建立。

```sql
CREATE [UNIQUE] INDEX indexName
ON TableName (columnName [ASC | DESC] [,...])

DROP INDEX indexName
```

### DML

#### 单表查询

```sql
SELECT       [DISTINCT | ALL] {* | [columnExpression [AS newName]] [,...]}
FROM         TableName [alias] [,...]
[WHERE       condition]
[GROUP BY    columnList] [HAVING condition]
[ORDER BY    columnList]
```

DISTINCT 用于去重。
‘*’ 用于查询所有字段。
AS 为字段命名。

#### 条件查询

WHERE 子句支持一下条件运算：

* 比较：+、<>、!=、<、>、<=、>=。
* 范围：BETWEEN AND、NOT BTWEEN AND，如 WHERE salary >= 20000 AND salary <= 30000。
* 成员关系：IN、NOT IN，如 WHERE position IN (‘Manager’, ‘Supervisor’)。
* 模式匹配：LIKE、NOT LIKE。模式匹配符号有：% 表示通配符，_ 任意单个字符。如 WHERE address LIKE ‘H%’ 表示匹配 H 起始的字符串。
* 空匹配：IS NULL、IS NOT NULL。

#### 排序

ORDER BY 子句可用于排序。保留字 ASC 表示升序，DESC 表示降序，默认使用升序。ORDER BY 子句支持多个字段排序，如 ORDER BY type, rent DESC。

#### 聚集函数

聚集函数可用于 SELECT 语句或 HAVING 子句中。当 SELECT 语句包含聚集函数时，那就不能在 SELECT 语句直接引用列字段，除非它作为聚集函数的参数。

* COUNT：返回指定列中数据的个数，能用于数值和非数值字段。
* SUM：返回指定列中数据的总和，只能用于数值字段。
* AVG：返回指定列中数据的平均值，只能用于数值字段。
* MIN：返回指定列中数据的最小值，能用于数值和非数值字段。
* MAX：返回指定列中数据的最大值，能用于数值和非数值字段。

```sql
SELECT COUNT(*) AS myCount FROM PropertyForRent WHERE rent > 350;
SELECT COUNT(DISTINCT propertyNo) AS myCount FROM Viewing WHERE viewDate BETWEEN '1-May-13' AND '31-May-13';
```

#### 分组

GROUP BY 用于分组查询，通常和聚集函数一样适用于报表。HAVING 子句用于过滤分组结果。

```sql
SELECT branchNo, COUNT(staffNo) AS myCount, SUM(salary) AS mySum
FROM Staff
GROUP BY branchNo
HAVING COUNT(staff) > 1
ORDER BY branchNo;
```

#### 多表查询

多表查询可使用子查询或连接操作。简单连接通过 WHERE 语句指定连接列。

简单连接

```sql
SELECT c.clientNo, fName, iName, propertyNo, comment
FROM Client c, Viewing v
WHERE c.clientNo = v.clientNo
```

JOIN 连接

```sql
--- 内连接，求交集，查找多表中值相同的行，剔除不相同的行
SELECT b.*, p.*
FROM Branch b, PropertyForRent p
WHERE b.bCity = p.pCity

--- 左外连接，求差集，保留左表中的所有行
SELECT b.*, p.*
FROM Branch b LEFT JOIN PropertyForRent p ON b.bCity = p.pCity

--- 右外连接，求差集，保留右表中的所有行
SELECT b.*, p.*
FROM Branch b RIGHT JOIN PropertyForRent p ON b.bCity = p.pCity

--- 全外连接，求并集，保留多表中的所有行
SELECT b.*, p.*
FROM Branch b RIGHT JOIN PropertyForRent p ON b.bCity = p.pCity
```

#### 嵌套查询

在某种意义上，分组查询是特殊的嵌套查询。子查询可用于 WHEHE、HAVING 子句，也可用于 INSERT、UPDATE、DELETE 语句。根据子查询的结果，子查询可以分为标量子查询（返回单行单列）、行子查询（返回单行多列）、表子查询（返回多行数据）。子查询可以添加 ANY、ALL、SOME、EXISTS、NOT EXISTS、NOT IN 等保留字，分别表示子查询获得的数据只要有任意一个、或全部满足条件即为真。

```sql
--- 等价于分组查询
SELECT branchNo, (SELECT COUNT(staffNo) AS myCount
                  FROM Staff s
                  WHERE s.branchNo = b.branchNO),
                  (SELECT SUM(staffNo) AS mySum
                  FROM Staff s
                  WHERE s.branchNo = b.branchNO)
FROM Branch b
ORDER BY branchNo;

--- 子查询作为 WHERE 条件
SELECT staffNo, fName, IName, position
FROM Staff
WHERE salary = (SELECT salary
                  FROM Staff
                  WHERE branchNo = 'B003');
```

#### 复合查询

复合查询指对多个 SELECT 语句求并集、交集、差集。

```sql
--- 求并集，两表的 city 列合并成一个新的查询结果
(SELECT city
FROM Branch
WHERE city IS NOT NULL)
UNION
(SELECT city
FROM PropertyForRent
WHERE city IS NOT NULL);

--- 求交集
(SELECT city
FROM Branch)
INTERSECT
(SELECT city
FROM PropertyForRent);

--- 求差集
(SELECT city
FROM Branch)
EXCEPT
(SELECT city
FROM PropertyForRent)
```

#### 更新数据

* INSERT：添加
* UPDATE：更新
* DELETE：删除

```sql
--- 添加
INSERT INTO TableName [(columnList)]
VALUES (dataValueList)

--- 复制数据
INSERT INTO TableName [(columnList)]
  SELECT ...

--- 更新，可同时更新多个列，也可以使用计算表达式
UPDATE TableName 
SET columnName1 = dataValue1 [, columnName2 = dataValue2 ...]
[WHERE searchCondition]

--- 删除，不指定条件时为删除全部数据
DELETE FROM TableName
[WHERE searchCondition]
```