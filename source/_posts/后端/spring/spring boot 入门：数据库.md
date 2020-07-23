---
title: spring boot 入门：数据库
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 63b17f08
date: 2019-03-17 03:00:00
updated: 2019-03-17 03:00:00
---

在 Spring Boot 中可使用的持久层访问框架 hibernate、mybatis，这里只介绍 mybatis。mybatis 使用有 xml 或注解两种方式。我们最常使用的是基于 xml 的方式，因此这里只介绍使用 xml 的方式。这里以 mysql 数据库为例，介绍一下如何使用 mybatis：

1. 添加 mysql-connector-java、mybatis-spring-boot-starter 依赖。
2. application.yml 中添加 datasource、mybatis 配置。
3. 编写实体类以及 mapper.java 文件，约定 service 层可调用的接口。
4. 编写 mapper.xml 文件，约定数据库访问操作。

application.yml 配置文件可如下设置：

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

# 基于 xml 配置添加
mybatis:
  mapper-locations: classpath:mapping/*Mapper.xml
  type-aliases-package: com.example.demo
```

mapper.xml 文件典型如下：

```xml
<!-- resources/mapper 中添加 mapper.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<!-- 指定 namespace -->
<mapper namespace="com.example.demo.dao.mapper.UserMapper">
    <!-- resultMap 用于设置返回值的类型和映射关系，并制定实体类 -->
    <resultMap id="BaseResultMap" type="com.example.demo.dao.domain.User" >
        <id column="id" property="id" jdbcType="BIGINT" />
        <result column="name" property="name" jdbcType="VARCHAR" />
    </resultMap>
    <sql id="Base_Column_List" >
        id, name
    </sql>
    <!-- id 与 Mapper 接口的方法名对应，同一个命名空间下不能重复，且不能带有 '.' 号 -->
    <select id="getUser" resultMap="BaseResultMap" parameterType="java.lang.Long">
        select 
        <include refid="Base_Column_List" />
        from user 
        where id = #{id,jdbcType=BIGINT}
    </select>

    <!-- useGeneratedKeys、keyProperty 为实体类注入 id 的值 -->
    <insert id="createUser" parameterType="com.example.demo.dao.domain.User" useGeneratedKeys="true" keyProperty="id">
        insert into user (
            id, name
        )
        values (
            #{id,jdbcType=BIGINT}, #{name,jdbcType=VARCHAR}
        )
    </insert>

    <update id="updateUser" parameterType="com.example.demo.dao.domain.User">
        update user
        set name = #{name,jdbcType=VARCHAR}
        where id = #{id,jdbcType=BIGINT}
    </update>

    <delete id="delete" parameterType="java.lang.Long">
        delete from user where id = #{id,jdbcType=BIGINT}
    </delete>
</mapper>
```

实体类以及 mapper 接口如下：

```java
@lombok
public class User implements Serializable {
    private static final long serialVersionUID = 1L; 

    private Integer id;
    private String name;
}

@Mapper
public interface MybatisUserAnnotationMapper {
    User getUser(@Param("id") Integer id);

    void createUser(User user);

    void updateUser(@Param("id") Integer id, @Param("name") String name);

    void delete(@Param("id") Integer id);
}
```

### mybatis-generator

mybatis-generator 能帮助开发者快速生成 mapper 文件，而不需要手动编写。

1. pom.xml 文件中添加 mybatis-generator-maven-plugin 编译插件。
2. 创建并配置 resources/generatorConfig.xml。
3. maven 面板点击 Plugins - mybatis-generator:genrate 以生成 mapper 文件。

pom.xml 追加配置如下：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.5</version>
            <configuration>
                <verbose>true</verbose>
                <overwrite>true</overwrite>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>org.mybatis.generator</groupId>
                    <artifactId>mybatis-generator-core</artifactId>
                    <version>1.3.2</version>
                </dependency>

                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.15</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

generatorConfig.xml 文件配置如下，声明数据库连接且需要访问的表：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="MysqlTables" targetRuntime="MyBatis3">

        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true"/>
            <!-- 将数据库中表的字段描述信息添加到注释 -->
            <property name="addRemarkComments" value="true"/>
            <!-- 是否生成注释代时间戳-->
            <property name="suppressDate" value="false" />
        </commentGenerator>

        <!-- jdbc链接信息 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&amp;characterEncoding=UTF8"
                        userId="root" password="123456">
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- 生成PO类的位置 -->
        <javaModelGenerator targetPackage="com.example.generate_mapper.dal.domain"
                            targetProject="src/main/java">
            <property name="enableSubPackages" value="false"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- mapper映射文件生成的位置 -->
        <sqlMapGenerator targetPackage="resources.mapper"
                         targetProject="src/main">
            <property name="enableSubPackages" value="false"/>
        </sqlMapGenerator>

        <!-- mapper接口生成的位置 -->
        <javaClientGenerator type="XMLMAPPER"
                             targetPackage="com.example.generate_mapper.dal.mapper"
                             targetProject="src/main/java">
            <property name="enableSubPackages" value="false"/>
        </javaClientGenerator>

        <!-- 指定要生成的表 -->
        <table tableName="test" domainObjectName="Test"
               enableCountByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false"
               enableUpdateByExample="false"></table>
    </context>
</generatorConfiguration>
```

#### 常见问题

1. Communications link failure 报错
    数据库端口配置错误等，参看 [“Communications link failure”错误](https://blog.csdn.net/qq_27471405/article/details/80921846)
2. Unknown system variable ‘query_cache_size’ 报错
    mysql 驱动程序版本与 mysql 服务器版本不一致，查看 [Unknown system variable ‘query_cache_size’](https://blog.csdn.net/qq_21870555/article/details/80711187)

### 分页

分页是应用中常见的数据库操作需求，一般实现可借助 limit, count 子句。[Mybatis-PageHelper](https://github.com/pagehelper/Mybatis-PageHelper) 提供了便利的分页脚本编写功能。以下是它的使用示例：

1. 添加 com.github.pagehelper 依赖。
2. application.yml 添加 pagehelper 配置。
3. services 层编写分页脚本。

```yml
pagehelper:
  helperDialect: mysql # 设置数据库方言
  reasonable: true # 分页合理化，即超过总条目时设为总条目数
  supportMethodsArguments: true # 支持通过 Mapper 接口参数来传递分页参数
  params: count=countSql # 设置参数映射
```

以下是分页脚本编写的[官方示例](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md)：

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

### druid

druid 作为数据库连接池，可产生监控记录。

1. 首先添加 druid 依赖。
2. 其次配置 application.yml 文件。
3. 最后添加 DruidConfig 文件。

访问 localhost:9090/druid/index.html 可查询监控数据。

```yml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai # useSSL=true 解决JDBC版本与MySQL版本不兼容问题
    username: odic
    password: Gw32_iuX
    druid:
      # 连接池配置(通常来说，只需要修改initialSize、minIdle、maxActive
      # 如果用Oracle，则把poolPreparedStatements配置为true，mysql可以配置为false。分库分表较多的数据库，建议配置为false。removeabandoned不建议在生产环境中打开如果用SQL Server，建议追加配置)
      initial-size: 5 # 初始化物理连接个数
      min-idle: 2 # 连接池中最小空闲连接数
      max-active: 20 # 连接池中最大的活跃连接数
      max-wait: 60000 # 配置获取连接等待超时的时间
      pool-prepared-statements: false # 开启缓存preparedStatement(PSCache)
      max-pool-prepared-statement-per-connection-size: -1 # 启用PSCache后，指定每个连接上PSCache的大小
      test-on-borrow: false # 申请连接时不检测连接是否有效
      test-on-return: false # 归还连接时不检测连接是否有效
      test-while-idle: true # 如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效
      validation-query: SELECT 'x'
      time-between-eviction-runs-millis: 60000 # 检测连接的间隔时间，若连接空闲时间 >= minEvictableIdleTimeMillis，则关闭物理连接
      min-evictable-idle-time-millis: 300000 # 连接保持空闲而不被驱逐的最小时间
      filter: config,stat # 配置监控统计拦截的filters（不配置则监控界面sql无法统计），监控统计filter:stat，日志filter:log4j，防御sql注入filter:wall
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000;config.decrypt=false; # 通过connectProperties属性来打开mergeSql功能；慢SQL记录
      web-stat-filter: 
        enabled: true # 是否启用StatFilter默认值true
        url-pattern: /*
        exclusions: /*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
        session-stat-enable: false
        session-stat-max-count: 1000
        principal-session-name: admin # 从session中获取身份标识
        principal-cookie-name: admin
        profile-enable: true
      # StatViewServlet配置
      #展示Druid的统计信息,StatViewServlet的用途包括：
      #   1.提供监控信息展示的html页面
      #   2.提供监控信息的JSON API
      stat-view-servlet:
        enabled: true # 是否启用StatViewServlet默认值true
        url-pattern: /druid/* # 根据配置中的url-pattern来访问内置监控页面
        reset-enable: false # 禁止手动重置监控数据
        login-username: admin # 设置监控页面的登录名和密码
        login-password: admin
        #StatViewSerlvet展示出来的监控信息比较敏感，是系统运行的内部情况，如果你需要做访问控制，可以配置allow和deny这两个参数
        #deny优先于allow，如果在deny列表中，就算在allow列表中，也会被拒绝。如果allow没有配置或者为空，则允许所有访问
        #配置的格式：<IP> 或者 <IP>/<SUB_NET_MASK_size>其中128.242.127.1/24
        #24表示，前面24位是子网掩码，比对的时候，前面24位相同就匹配,不支持IPV6。
        #deny:xxx
        #allow: 127.0.0.1
      aop-patterns: com.example.demo.*.*.* # 对内部各接口调用的监控
```

```java
@Configuration
public class DruidConfig {
    private Logger log = Logger.getLogger(LogAspect.class);

    @Bean
    @ConfigurationProperties("spring.datasource")
    public DataSource druidDataSource() {
        return new DruidDataSource();
    }

    @Bean
    public ServletRegistrationBean druidStatViewServlet() {
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(new StatViewServlet(), "/druid/*");
        registrationBean.addInitParameter("allow", "127.0.0.1");// IP白名单 (没有配置或者为空，则允许所有访问)
        registrationBean.addInitParameter("deny", "");// IP黑名单 (存在共同时，deny优先于allow)
        registrationBean.addInitParameter("loginUsername", "admin");
        registrationBean.addInitParameter("loginPassword", "123456");
        registrationBean.addInitParameter("resetEnable", "false");
        return registrationBean;
    }

    @Bean
    public FilterRegistrationBean druidWebStatViewFilter() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean(new WebStatFilter());
        registrationBean.addInitParameter("urlPatterns", "/*");
        registrationBean.addInitParameter("exclusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");
        return registrationBean;
    }
}
```

### lombok 及 Serializable

lombok 提供的注解可用于简化实体类的编写，使用时须引入 lombok 依赖。在 IDEA 环境使用 lombok，还需要添加 lombok 插件，随后在 file - setting 面板启动 AnnocationProcessors。更多内容可戳 [Lombok的基本使用](https://www.jianshu.com/p/2543c71a8e45)。

一般而言，实体类还需要继承 Serializable 接口，以便通过 ObjectOutputStream、ObjectInputStream 实现对象和字节流的相互转换。在 IDEA 环境中，还需要 setting - Inspections 面板上选中 Serializable class without 'serialVersionUID'，以便使用 alt + enter 快捷键生成 serialVersionUID。更多内容可戳 [Java对象为啥要实现Serializable接口？](https://developer.51cto.com/art/201905/596334.htm)、[IDEA给Serializable类加上自动的serialVersionUID](https://www.cnblogs.com/davidgu/p/9501776.html)。

### 事务

当一个请求中组织了多表操作行为，某个表操作失败也将使前续操作变得无效。这时候，就可以使用事务机制进行回滚。简单的，为方法、类添加 @Transactional 注解，就可以声明事务。需注意的是，@Transactional 注解只能使用 public 方法上，且当脚本抛出异常时，事务才生效。

```java
@Transactional
public void doDbOperation() throws Exception {
    // do db operation
}
```