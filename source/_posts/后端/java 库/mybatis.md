---
title: mybatis
category:
  - 后端
  - java
tags:
  - 后端
  - java
keywords: 'JDBC,java,mybatis'
abbrlink: c72b6c8d
date: 2019-11-23 00:00:00
updated: 2019-11-23 00:00:00
---

按我的理解，可以在现代前端技术找到与 Mybatis 相类的解决方案。虽然浏览器开列了 DOM API 用于操纵节点，但是存在以下两个主要问题：不同的浏览器使用不同的接口；操纵节点的开发成本高昂。jQuery 等节点操作型类库解决了第一个问题，但没有解决第二个问题。到了模板引擎 + 虚拟 dom 的技术实现方案后，由虚拟 dom 统一封装节点操纵接口，并作性能优化，再由模板引擎对接虚拟 dom，前端只需要关心模板，达到声明式编程的效果。话题切回到 Mybatis，在此之前，JDBC 解决了不同数据库的对接问题，但是需要反复的创建和关闭 Statement、ResultSet；sql 语句散落，且不能复用，频繁书写对象关系模型的转换逻辑；动态 sql 以参数序号拼接，造成它与业务逻辑有较高的耦合度等。针对这些问题，我们能想象到的处理方式是：抽象 sql 执行流程；使用模板描述 sql 语句。前者的意义就在于组装 sql 操作的流水线工厂，包含但不限于：创建 Connection，根据 sql 预备语句创建 PreparedStatement，通过关系对象模型获得参数，执行 sql，将结果转换成关系对象模型，关闭 Connection 等。可以说，JDBC 的焦点在于流水线的某个节点，Mybatis 的焦点在于整条流水线，这样一来，应用开发者就不大需要关心流水线上的某个节点该怎么操作，而只需关心整条流水线的输入输出（当然，若要关心流水线上的特定节点，也可以使用钩子或拦截器等手段处理）。有了流水线工厂，输入等表现形式可以是多样的，正如 Mybatis 既可以使用 xml 配置声明待执行的 sql 语句，也可以使用 Java 语句约定该执行的 sql 语句。在使用模板约定 sql 语句的基础上，sql 语句是既可以组装和复用的，又会特定的对象关系模型达成相互转换关系。

![image](mybatis1.png)

上述 Mybatis 架构图中，Mybatis 首先会从 MybatisConfig.xml 加载基本的配置信息，包含环境信息、数据库的驱动程序、数据库地址、账号和密码等，构建 Configuration 对象；然后注册 Mapper.xml 映射器，Mapper.xml 描述了待执行的 sql、及其与关系对象模型的映射关系等。在取得配置信息以后，Mybatis 会通过 SqlSessionFactoryBuilder 创建 SqlSessionFactory；sqlSessionFactory 保有 configuration 配置信息；SqlSessionFactory 允许使用 sqlSessionFactory.openSession 方法创建 SqlSession（openSession 方法用于设定 sqlSession 的自动提交模式、执行模式等）。因为全量映射器信息已经存在内存中，SqlSession 允许我们直接使用 sqlSession.getMapper 形式获取映射器的可用形态，继而执行映射器中约定的方法；使用 sqlSession.select 能直接执行颗粒度更细的映射器方法。SqlSessionFactory 的最佳作用域是应用作用域；SqlSession 不是线程安全的，不能作为一个类的静态属性或实例属性，每当有请求时才予以创建，且需要及时关闭。与上图表述不尽相同的是，在创建 SqlSession 属性的过程中，Mybatis 会根据 execType 选用不同的 Exector 实现类，Exector 实例本身作为 SqlSession 实例的一个属性。当我们调用 sqlSession.select 时，Mybatis 首先会从 configuration 中选用相关的 MappedStatement，然后调用 executor.query，由 exector 实际消费 MappedStatement 作出入参数转换。MappedStatement 通过解析 Mapper.xml 获得，它约定了 sql 语句的处理方式。在 executor.query 执行期间，Mybatis 会通过 MappedStatement 获得 StatementHandler；StatementHandler 用于预处理 statement，执行实际的 sql 语句。

![image](mybatis2.png)

以下是 sqlSession.select 的执行逻辑。

```java
public class DefaultSqlSession implements SqlSession {
  @Override
  public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
}

public abstract class BaseExecutor implements Executor {
  // BoundSql 用于获取真实的 sql 语句 
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }

  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
}

public class SimpleExecutor extends BaseExecutor {
  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      // 获取顶层 Configuration 配置信息
      Configuration configuration = ms.getConfiguration();
      // 获取具体的 StatementHandler，如 SimpleStatement，PreparedStatement 或 CallableStatement
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      // 创建 statement
      stmt = prepareStatement(handler, ms.getStatementLog());
      // 执行实际的 sql，并处理结果集
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
}

public class Configuration {
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 根据 ms.getStatementType 选用具体的 StatementHandler
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 使用拦截器插件对 StatementHandler 进行封装
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
}

public abstract class BaseStatementHandler implements StatementHandler {
  protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    this.configuration = mappedStatement.getConfiguration();
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;

    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();

    if (boundSql == null) { // issue #435, get the key before calculating the statement
      generateKeys(parameterObject);
      // 根据动态参数输出实际的 sql，实现见下文
      boundSql = mappedStatement.getBoundSql(parameterObject);
    }

    this.boundSql = boundSql;

    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }

  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
}

public class SimpleStatementHandler extends BaseStatementHandler {
  // 使用实际的 sql 连接 Connection 创建 statement
  @Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    if (mappedStatement.getResultSetType() != null) {
      return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.createStatement();
    }
  }

  // 执行实际的 sql，并处理结果集
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.<E>handleResultSets(statement);
  }
}
```

上文更多的是简要地表明了 sqlSession.select 发起的一个查询过程，这个过程相当于前端 handlebars 模板引擎的第二个阶段 —— 通过解析函数处理参数并获得实际的 html。可是 handlebars 还有第一个阶段 —— 通过模板获得解析函数。两个阶段拼合在一起，才合成一个完成的过程 const parse = Handlebars.compile(template); const html = parse(data)。与此相同，Mybatis 同样需要先将 Mapper.xml 配置转换成 MappedStatement，这样才能通过 MappedStatement 获得 BoundSql，再通过 BoundSql 获得实际待执行的 sql 语句。下图既表明了 Mybatis 的架构，也说明了解析和执行正是虎符的两半。

![image](mybatis3.png)

通过回溯源码，我们发现在 spring 工程中，Mybatis 利用 spring 的机制加载 Mapper.xml 文件。正是在 org.mybatis.spring 包调用了 xmlMapperBulder.parse 方法，configuration 中才会有 XMLStatementBuilder 实例。而通过 XMLStatementBuilder 实例解析 xml 文件中的相关节点，MappedStatement 实例才得以被添加到 configuration 中。同样采用回溯法，我们发现，BoundSql 的创建过程依循这样一条路线：在 xml 解析过程中，会创建 xmlScriptBuilder 实例，再由 xmlScriptBuilder 解析 xml 配置中的 sql 语句节点，获得 rawSqlSource 或 dynamicSqlSource 实例（两者的 getBoundSql 方法即可用于获得 boundSql 实例）。以下源码仅以简要说明 mappedStatement.getBoundSql(parameterObject) 方法为什么能完成动态参数的拼接。

```java
public class RawSqlSource implements SqlSource {
  // rootSqlNode 为 sql 语句节点
  public RawSqlSource(Configuration configuration, SqlNode rootSqlNode, Class<?> parameterType) {
    this(configuration, getSql(configuration, rootSqlNode), parameterType);
  }

  public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<String, Object>());
  }
}

public class DynamicSqlSource implements SqlSource {
  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }

}

public class SqlSourceBuilder extends BaseBuilder {
  public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql = parser.parse(originalSql);
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
}

// 将 "#{", "}" 中间替换以动态参数
public class GenericTokenParser {
  public String parse(String text) {
    if (text == null || text.isEmpty()) {
      return "";
    }
    // search open token
    int start = text.indexOf(openToken, 0);
    if (start == -1) {
      return text;
    }
    char[] src = text.toCharArray();
    int offset = 0;
    final StringBuilder builder = new StringBuilder();
    StringBuilder expression = null;
    while (start > -1) {
      if (start > 0 && src[start - 1] == '\\') {
        // this open token is escaped. remove the backslash and continue.
        builder.append(src, offset, start - offset - 1).append(openToken);
        offset = start + openToken.length();
      } else {
        // found open token. let's search close token.
        if (expression == null) {
          expression = new StringBuilder();
        } else {
          expression.setLength(0);
        }
        builder.append(src, offset, start - offset);
        offset = start + openToken.length();
        int end = text.indexOf(closeToken, offset);
        while (end > -1) {
          if (end > offset && src[end - 1] == '\\') {
            // this close token is escaped. remove the backslash and continue.
            expression.append(src, offset, end - offset - 1).append(closeToken);
            offset = end + closeToken.length();
            end = text.indexOf(closeToken, offset);
          } else {
            expression.append(src, offset, end - offset);
            offset = end + closeToken.length();
            break;
          }
        }
        if (end == -1) {
          // close token was not found.
          builder.append(src, start, src.length - start);
          offset = src.length;
        } else {
          builder.append(handler.handleToken(expression.toString()));
          offset = end + closeToken.length();
        }
      }
      start = text.indexOf(openToken, offset);
    }
    if (offset < src.length) {
      builder.append(src, offset, src.length - offset);
    }
    return builder.toString();
  }
}

public class StaticSqlSource implements SqlSource {
  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Configuration configuration;

  public StaticSqlSource(Configuration configuration, String sql) {
    this(configuration, sql, null);
  }

  public StaticSqlSource(Configuration configuration, String sql, List<ParameterMapping> parameterMappings) {
    this.sql = sql;
    this.parameterMappings = parameterMappings;
    this.configuration = configuration;
  }

  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
  }
}
```

### 后记

因为半路出家的缘故，我很难用精准的词汇去描述所思所想，表达也就会大打折扣。很多时候，我想把编程比喻为整理抽屉，需要把东西摆放得足够整齐，需要知道哪个格子藏了什么东西（后一条会显得很有经验）。但是茴香豆有几种写法，Mybatis 设计了多少种 xml 配置项，这不是我所关心的重点。

### 参考

[Mybatis 文档](https://mybatis.org/mybatis-3/zh/getting-started.html)
[MyBatis的工作原理](http://c.biancheng.net/view/4304.html)
[终结篇：MyBatis原理深入解析](https://www.jianshu.com/p/ec40a82cae28)
[Mybatis架构与原理](https://www.jianshu.com/p/15781ec742f2)