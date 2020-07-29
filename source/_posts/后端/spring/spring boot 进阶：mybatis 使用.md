---
title: spring boot 进阶：mybatis 使用
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring,mybatis
abbrlink: a80eb170
date: 2020-05-04 06:00:00
updated: 2020-05-04 06:00:00
---

本篇内容基于《mybatis 从入门到精通》整理。

### 基本使用

pom.xml 添加 mybatis-spring-boot-starter 依赖。

编写 Mapper.xml、实体类、Mapper 接口等。

```java
@data
class Test {
  private Integer id;
  private String name;
  private String description;
}

@Mapper
public interface TestMapper {
  Test selectByPrimaryKey(Long id);
}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<!-- 指定 namespace -->
<mapper namespace="com.example.demo.dao.mapper.TestMapper" >
  <!-- resultMap 用于设置返回值的类型和映射关系 -->
  <resultMap id="BaseResultMap" type="com.example.demo.dao.domain.TestDO" >
    <id column="id" property="id" jdbcType="BIGINT" />
    <result column="name" property="name" jdbcType="VARCHAR" />
    <result column="description" property="description" jdbcType="VARCHAR" />
    <result column="gmt_create" property="gmtCreate" jdbcType="TIMESTAMP" />
    <result column="create_nick" property="createNick" jdbcType="VARCHAR" />
    <result column="gmt_modified" property="gmtModified" jdbcType="TIMESTAMP" />
    <result column="modified_nick" property="modifiedNick" jdbcType="VARCHAR" />
  </resultMap>
  <sql id="Base_Column_List" >
    id, name, description, 
    gmt_create, create_nick, gmt_modified, modified_nick
  </sql>
  <!-- id 与 Mapper 接口的方法名对应，同一个命名空间下不能重复，且不能带有 '.' 号 -->
  <select id="selectByPrimaryKey" resultMap="BaseResultMap" parameterType="java.lang.Long" >
    select 
    <include refid="Base_Column_List" />
    from test
    where id = #{id,jdbcType=BIGINT}
  </select>
</mapper>
```

### mapper.xml

#### resultMap

resultMap 标签用于设置返回值的类型和映射关系。resultMap 包含如下属性：id 必填且唯一；type 对应 java 实体类；extends 指定扩展关系；autoMapping 是否开启自动映射功能。

resultMap 可设置以下标签：

* constructor：使用构造函数创建结果。idArg 子标签用于指定 id；arg 指定其余参数。
* id：标记结果作为 id，可以提高整体性能。其包含属性有：column 列名；property 对应实体类中的属性，支持 namepath；javaType 实体类中的类型别名（JavaBean 可自动判断类型；如映射到 HashMap 须指定）；jdbcType 列字段的数据库类型；typeHandler 指定类型处理器。result 标签同样含有这些属性。
* result：注入到实体类中的普通结果。
* association：一个复杂的类型关联，许多结果将包成这种类型。
* collection：复杂类型的集合。
* discriminator：根据结果值来决定使用哪个结果映射。
* case：基于某些值的结果映射。
* resultMap 的 id 可以作为 select 标签的 resultMap 属性值。除了 resultMap 标签以外，select 标签的 resultMap 属性还可以设置为 java 实体类。

#### select

select 标签用于查询。

```xml
<!-- 嵌套属性赋值 -->
<select id="selectUserAndRoleById" resultType="tk.mybatis.simple.model.SysUser">
  select u.id,
    u.user_name userName, u.user_password userPassword, u.user_email userEmail ,
    u.user_info userinfo, u.head_img headimg, u.create_time createTime, 
    r.id "role.id", r.role_name "role.roleName", 
    r.enabled "role.enabled", 
    r.create_by "role.createBy",
    r.create_time "role.createTime"
  from sys_user u
    inner join sys_user_role ur on u.id = ur.user_id 
    inner join sys_role r on ur.role_id = r .id
    where u.id = #{id}
</select>
```

#### insert

insert 标签用于插入。insert、update 可使用如下属性：

* parameterType：设置传入的参数。mybatis 会自动推断传入的类型，因此不建议设置。
* flushCache：当调用发生时是否清空一级缓存和二级缓存。
* timeout：超时时间。
* statementType：对于 STATEMENT、 PREPARED、 CALLABLE, MyBatis会分别使用 对应的 Statement 、 PreparedStatement 、 CallableStatement ，默认 值为 PREPARED 。
* useGeneratedKeys：是否使用 JDBC 的 getGeneratedKeys 方法获取主键，默认为 false。
* keyProperty：设置获取主键值后将要赋值的属性名。
* keyColumn：通过生成的键值设置表中的列名，仅在某些数据库（如 PostgreSQL）中是必须的。
* databaseId：如果配置了 databaseidProvider(4.6节有详细配置方法)， MyBatis 会加载所有的不带 databaseid 的或匹配当前 databaseid 的语句 。如果同时 存在带 databaseid 和不带 databaseid 的语句，后者会被忽略。

```xml
<insert id="insert" parameterType="com.example.demo.dao.domain.TestDO" useGeneratedKeys="true" keyProperty="id">
  insert into test (
    id, name, description, 
    gmt_create, create_nick, gmt_modified, modified_nick
  )
  values (
    #{id,jdbcType=BIGINT}, #{name,jdbcType=VARCHAR}, #{description,jdbcType=VARCHAR}, 
    #{gmtCreate,jdbcType=TIMESTAMP}, #{createNick,jdbcType=VARCHAR}, 
    #{gmtModified,jdbcType=TIMESTAMP}, #{modifiedNick,jdbcType=VARCHAR}
  )
</insert>

<!-- 使用 selectKey 返回主键 -->
<insert id="insert" parameterType="com.example.demo.dao.domain.TestDO" useGeneratedKeys="true" keyProperty="id">
  insert into test (
    id, name, description, 
    gmt_create, create_nick, gmt_modified, modified_nick
  )
  values (
    #{id,jdbcType=BIGINT}, #{name,jdbcType=VARCHAR}, #{description,jdbcType=VARCHAR}, 
    #{gmtCreate,jdbcType=TIMESTAMP}, #{createNick,jdbcType=VARCHAR}, 
    #{gmtModified,jdbcType=TIMESTAMP}, #{modifiedNick,jdbcType=VARCHAR}
  )
  <selectKey keyColumn="id" resultType="long" keyProperty="id" order="AFTER">
    SELECT LAST_INSERT_ID() 
  </selectKey>
</insert>
```

#### update

update 标签用于更新。

```xml
<update id="updateByPrimaryKey" parameterType="com.example.demo.dao.domain.TestDO" useGeneratedKeys="true" keyProperty="id">
  update test
  set name = #{name,jdbcType=VARCHAR},
    description = #{description,jdbcType=VARCHAR},
    gmt_create = #{gmtCreate,jdbcType=TIMESTAMP},
    create_nick = #{createNick,jdbcType=VARCHAR},
    gmt_modified = #{gmtModified,jdbcType=TIMESTAMP},
    modified_nick = #{modifiedNick,jdbcType=VARCHAR}
  where id = #{id,jdbcType=BIGINT}
</update>
```

#### delete

delete 标签用于删除。

```xml
<delete id="deleteByPrimaryKey" parameterType="java.lang.Long" >
  delete from test
  where id = #{id,jdbcType=BIGINT}
</delete>
```

#### 多参数

使用 @Param 注解可为 insert、update 语句设置多个实体类作为参数。

#### 动态 sql

```xml
<!-- if 条件 -->
<if test="name != null and name != ''"> 
  and name like concat('%', #{name}, '%') 
</if>

<!-- choose 条件 -->
<choose>
  <when test="id != null">
    and id = #{id}
  </when >
  <when test="name != null and name != ''"> 
    and name = #{name}
  </when> 
  <otherwise> 
    and 1 = 2 
  </otherwise>
</choose>

<!-- where 条件，添加一个 where 子句，它会自动剔除 and -->
<where>
  <if test="userName != null and userName != ''"> 
    and user_name like concat('%', #{userName}, '%') 
  </if>
  <if test="userEmail != '' and userEmail != null"> 
    and user_email = #{userEmail}
  </if>
</where>

<!-- set 标签，添加一个 set 子句 -->
<set>
  <if test="name != null and name !=''">
    name = #{name} ,
  </if>
  id = #{id},
</set>

<!-- 使用 trim 标签实现 where -->
<trim prefix="WHERE" prefixOverrides="AND"> 
  <!-- ... -->
</trim>

<!-- 使用 trim 标签实现 set -->
<trim prefix="SET" suffixOverridess=","> 
  <!-- ... -->
</trim>

<!-- foreach，collection 属性指定待循环的属性名；separator 分隔符；open、close 起始和结尾字符串 -->
<foreach collection="list" item="user" separator=",">
  #{user.name}, #{user.description}, #{user.createNick}, 
  #{user.gmtCreate, jdbcType=TIMESTAMP})
</foreach>

<!-- 使用 bind 创建 OGNL 表达式 -->
<if test="name != null and name != ''">
  <bind narne="nameLike" value="'%' + name + '%'" /> 
  and name like #{nameLike}
</if>
```

### 注解

#### 基本注解

基本注解包含 @Results、@Select、@Insert、@Update、@Delete。

```java
// 查询
@Select({"
  select id, name, description,
  gmt_create gmtCreate, create_nick createNick, 
  gmt_modified gmtModified, modified_nick modifiedNick
  from test
  where id = #{id}"
}) 
Test selectById(Long id);

// 使用 resultMap
@Results({ 
  @Result(property = "id", column = "id", id = true),
  @Result(property = "name", column = "name"),
  @Result(property = "description", column = "description"),
  @Result(property = "gmtCreate", column = "gmt_create"),
  @Result(property = "createNick", column = "create_nick"),
  @Result(property = "gmtModified", column = "gmt_modified"),
  @Result(property = "modifiedNick", column = "modified_nick"),
})
@Select({"
  select id, name, description,
  gmt_create, create_nick, gmt_modified, modified_nick
  from test
  where id = #{id}"
}) 
Test selectById(Long id);

// 插入
@Insert({"insert into test (name, description, create_nick, gmt_create)", 
  "values(#{name}, #{description}, #{createNick}, #{gmtCreate, jdbcType=TIMESTAMP})"
}) 
@Options(useGeneratedKeys = true, keyProperty = "id")
int insert(Test test);

// 更新
@Update({"update test",
  "set name = #{name}, description = #{description},", 
  "modified_nick = #{modifiedNick}, gmt_modified = #{gmtModified, jdbcType=TIMESTAMP}",
  "where id = #{id}" 
})
int updateById{Test test);

// 删除
@Delete("delete from test where id = #{id}")
int deleteById{Long id);
```

#### Provider 注解

Provider 注解有四种：@SelectProvider、@InsertProvider、@UpdateProvider、@DeleteProvider。

```java
@SelectProvider(type = PrivilegeProvider.class, method = "selectById") 
TestPrivilege selectById(Long id);

public class PrivilegeProvider {
  public String selectById(final Long id) {
    return new SQL(){
      SELECT("id, name, description");
      FROM("test");
      WHERE("id = #{id}");
    }.toString();
  }
}
```