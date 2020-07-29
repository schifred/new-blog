---
title: spring boot 进阶：mybatis 通用 mapper
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring,mybatis
abbrlink: ebd608d4
date: 2020-05-03 06:00:00
updated: 2020-05-03 06:00:00
---

[Mybatis-Mapper 插件](https://github.com/abel533/Mapper) 用于简化 mybatis 使用，这里只介绍基本使用和代码生成，扩展通用 Mapper 接口和 Example 的使用可参考[官方文档](https://github.com/abel533/Mapper/wiki/5.extend)。

### 基本使用

1. pom.xml 添加 tk.mybatis 依赖。
2. 配置 application.yml，设置通用 Mapper（支持自定义）。
3. 编写 Mapper.xml。
4. 编写实体类，实体类中字段名默认按驼峰式进行转换（通过 @NameStyle 注解可改变转换方式）。
5. 以继承通用 Mapper 的方式编写 Mapper 接口。

基于 Mapper 接口编写 Service 实现类。

application.yml 配置示例，配置项可参考 [文档](https://github.com/abel533/Mapper/wiki/3.config)：

```yml
mapper:
  notEmpty: true # insertSelective 和 updateByPrimaryKeySelective 是否判断字符串类型 !=''
Mapper.xml 编写示例：
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.dao.mapper.CountryMapper">
    <resultMap id="BaseResultMap" type="com.example.demo.dao.mapper.CountryMapper">
        <id column="Id" jdbcType="INTEGER" property="id" />
        <result column="countryname" jdbcType="VARCHAR" property="countryname" />
        <result column="countrycode" jdbcType="VARCHAR" property="countrycode" />
    </resultMap>
</mapper>
```

实体类和 Mapper 接口编写示例：

```java
// 实体类
import javax.persistence.Id;

public class Country {
  @Id
  private Integer id;
  private String  countryname;
  private String  countrycode;

  //省略 getter 和 setter
}

// Mapper 接口
import org.apache.ibatis.annotations.Select;
import tk.mybatis.mapper.common.Mapper;

public interface CountryMapper extends Mapper<Country> {
    @Select("select * from country where countryname = #{countryname}")
    Country selectByCountryName(String countryname);
}

// Service 实现类
import tk.mybatis.mapper.entity.Example;

@Service
public class CountryServiceImpl implements CountryService {
    @Autowired
    private CountryMapper countryMapper;

    public Country select(int id){
        Example example = new Example(Country.class);
        Example.Criteria criteria = example.createCriteria();
        criteria.andEqualTo("id", id);
        return countryMapper.selectOneByExample(example);
    }
}
```

#### 实体类可用注解

* @NameStyle：字段名到列名的映射方式，如 @NameStyle(Style.uppercase)。
* @Table：设置实体类关联的表名、数据库名。
* @Column：设置字段与哪个列关联，insertable、updateable 属性用于影响 sql 的插入、更新操作。
* @ColumnType：作用与 @Column 相似，优先级较低，可设置 jdbcType、类型处理器等。
* @Transient：说明不是表中的字段，List、Map 等属性不需要添加该注解。
* @Id：说明主键，联合主键可以配置多次，一个表中必须有一个主键。
* @KeySql：设置主键策略，用于替换 @GeneratedValue。如 @KeySql(useGeneratedKeys = true) 用于使 sql 操作返回主键等。参看 [文档](https://github.com/abel533/Mapper/wiki/2.3-generatedvalue)。
* @GeneratedValue：设置主键策略。
* @Version：实现乐观锁。

### 代码生成器

这里介绍的是通过 maven 插件生成代码。

1. pom.xml 添加 org.mybatis.generator 等 maven 插件。
2. 添加 resources/generator/generatorConfig.xml 配置文件。

```xml
<!-- pom.xml -->
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <source>${java.version}</source>
    <target>${java.version}</target>
  </configuration>
</plugin>
<plugin>
  <groupId>org.mybatis.generator</groupId>
  <artifactId>mybatis-generator-maven-plugin</artifactId>
  <version>1.3.6</version>
  <configuration>
    <configurationFile>
      ${basedir}/src/main/resources/generator/generatorConfig.xml
    </configurationFile>
    <overwrite>true</overwrite>
    <verbose>true</verbose>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.29</version>
    </dependency>
    <dependency>
      <groupId>tk.mybatis</groupId>
      <artifactId>mapper</artifactId>
      <version>4.0.0</version>
    </dependency>
  </dependencies>
</plugin>

<!-- generatorConfig.xml -->
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <context id="Mysql" targetRuntime="MyBatis3Simple" defaultModelType="flat">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>

        <plugin type="tk.mybatis.mapper.generator.MapperPlugin">
            <!-- 设置通用 Mapper 接口 -->
            <property name="mappers" value="tk.mybatis.mapper.common.Mapper"/>
            <!-- 是否区分大小写 -->
            <property name="caseSensitive" value="true"/>
        </plugin>

        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/demo"
                        userId="root"
                        password="123456">
        </jdbcConnection>

        <javaModelGenerator targetPackage="com.example.demo.dao.entity"
                            targetProject="src/main/java"/>

        <sqlMapGenerator targetPackage="mapper"
                         targetProject="src/main/resources"/>

        <javaClientGenerator targetPackage="com.example.demo.dao.mapper"
                             targetProject="src/main/java"
                             type="XMLMAPPER"/>

        <table tableName="mapper_test">
            <generatedKey column="name" sqlStatement="JDBC"/>
        </table>
    </context>
</generatorConfiguration>
```