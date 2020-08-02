---
title: spring boot 入门：附录（三） mysql
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: b600fbee
date: 2019-03-17 13:00:00
updated: 2019-03-17 13:00:00
---

### 安装启动
mac 可使用 homebrew 安装 mysql，更多可戳 [使用Homebrew安装mysql](https://www.jianshu.com/p/d3f7e7402449)。

```bash
brew install mysql # 安装
mysql.server start # 启动
mysql.server stop # 暂停
mysql.server restart # 重启
```

安装并启动完成后，就可以使用客户端 navicat、mysql workbench 访问了。

### 建表

```sql
drop table user;
create table if not exists user(
   id bigint(20) unsigned not null auto_increment comment '主键',
   user_name varchar(20) not null comment '用户名',
   password varchar(20) not null comment '密码',
   mobile varchar(11) not null comment '手机号',
   email varchar(50) not null comment '邮箱',
  
     gmt_create TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP comment '创建时间',
   create_nick varchar(30) comment '创建人',
   gmt_modified TIMESTAMP default CURRENT_TIMESTAMP on update CURRENT_TIMESTAMP comment '修改时间',
   modified_nick varchar(30) comment '修改人',
   primary key (id)
) ENGINE=InnoDB AUTO_INCREMENT=1000 DEFAULT CHARSET=utf8 COMMENT='用户表';
```

#### 插数据

```sql
insert into user (user_name,password,mobile,email) values ('张三','123456','18888888888','zhangsan@test.com');
insert into user (user_name,password,mobile,email) values ('李四','123456','18888888888','lisi@test.com');
insert into user (user_name,password,mobile,email) values ('王五','123456','18888888888','wangwu@test.com');
```

#### 加字段

```sql
alert table user add column age int comment '年龄';
```