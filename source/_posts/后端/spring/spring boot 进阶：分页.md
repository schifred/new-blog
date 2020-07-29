---
title: spring boot 进阶：分页
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring,mybatis
abbrlink: 85e2f6c4
date: 2020-05-02 06:00:00
updated: 2020-05-02 06:00:00
---

### sql 分页

sql 分页借助于 limit 子句，这时应用层只需将前端页面上的参数传入数据库即可。

```sql
select * from tableName limit i, n --- i 偏移量，n 每页显示条数
select count(id) from tableName --- 获取总数
```

### 内存分页

内存分页即取出表中的所有数据，然后根据前端传入的页码和每页显示条数计算当前需要展示的数据。

```java
int count = 0;
if(list != null && list.size() > 0) {
	count = list.size();
	int fromIndex = current * pageSize;
	int toIndex = (current + 1) * pageSize;
	if(toIndex > count) {
		toIndex = count;
	}
	List<XDTO> data = list.subList(fromIndex, toIndex);
}
```

### Mybatis-PageHelper 插件

[Mybatis-PageHelper](https://github.com/pagehelper/Mybatis-PageHelper) 分页插件的使用步骤如下：

1. pom.xml 添加 com.github.pagehelper 依赖。
2. 配置 application.yml。
3. 编写 services。

```yml
pagehelper:
  helperDialect: mysql # 设置数据库方言
  reasonable: true # 分页合理化，即超过总条目时设为总条目数
  supportMethodsArguments: true # 支持通过 Mapper 接口参数来传递分页参数
  params: count=countSql # 设置参数映射
```

service 编写的官方示例：

```java
// 方法 1
PageHelper.startPage(1, 10);
List<User> list = userMapper.selectIf(1);// 使用拦截器从 PageHelper 中获取分页参数

// 方法 2
Page<User> page = PageHelper.startPage(1, 10).doSelectPage(()-> userMapper.selectGroupBy());

// 方法 3
PageInfo pageInfo = PageHelper.startPage(1, 10).doSelectPageInfo(() -> userMapper.selectGroupBy());

int total = PageHelper.count(()->userMapper.selectLike(user));
```