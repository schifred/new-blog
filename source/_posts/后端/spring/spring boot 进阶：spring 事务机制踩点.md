---
title: spring boot 进阶：spring boot 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 7c37d32a
date: 2019-12-29 06:00:00
updated: 2019-12-29 06:00:00
---

### what

spring 事务机制提供了统一的编程模型来处理不同数据访问操作（local transactions by using JDBC, JPA, or Hibernate），同时支持分布式事务（JTA transactions）；支持声明式事务编程和编程式事务编程，声明式事务编程对代码无侵入性，只需要改变配置类即可应对不同的数据访问技术；与 spring 数据访问抽象完美集成。对于 spring boot 项目，只要显式地对启动类添加 @EnableTransactionManagement 注解，即能为应用注入 PlatformTransactionManager 实例；然后在需要事务支持的方法或类上添加 @Transactional 注解（该注解推荐使用在类和类的公共方法上。如果在保护方法和私有方法上使用，需要配合 AspectJ），spring 就能基于 AOP 机制开启一个事务，当调用无异常时，事务就会被提交了。

### sample

#### basic

样例来自 [Managing Transactions](https://spring.io/guides/gs/managing-transactions/)，基于 spring boot 作了改造：

```java
@EnableTransactionManagement // 启注解事务管理
@SpringBootApplication
public class ProfiledemoApplication {
  @Bean
  public Object testBean(PlatformTransactionManager platformTransactionManager){
    System.out.println(">>>>>>>>>>" + platformTransactionManager.getClass().getName());
    return new Object();
  }

  public static void main(String[] args) {
    SpringApplication.run(ProfiledemoApplication.class, args);
  }
}

@Component
public class BookingService {
  private final static Logger logger = LoggerFactory.getLogger(BookingService.class);

  private final JdbcTemplate jdbcTemplate;

  public BookingService(JdbcTemplate jdbcTemplate) {
    this.jdbcTemplate = jdbcTemplate;
  }

  @Transactional
  public void book(String... persons) {
    for (String person : persons) {
      logger.info("Booking " + person + " in a seat...");
      jdbcTemplate.update("insert into BOOKINGS(FIRST_NAME) values (?)", person);
    }
  }

  public List<String> findAllBookings() {
    return jdbcTemplate.query("select FIRST_NAME from BOOKINGS",
        (rs, rowNum) -> rs.getString("FIRST_NAME"));
  }
}
```

#### multiple management

当应用中有多种数据库连接方式时，[Spring Boot的事务管理注解@EnableTransactionManagement的使用](https://blog.csdn.net/u010963948/article/details/79208328) 这篇文章指明了该如何使用声明式事务：

```java
@EnableTransactionManagement
@SpringBootApplication
public class ProfiledemoApplication implements TransactionManagementConfigurer {// 实现 TransactionManagementConfigurer，指定事务管理器
  @Resource(name="txManager2")
  private PlatformTransactionManager txManager2;

  // 创建事务管理器1，以数据源 DataSource 为参数
  @Bean(name = "txManager1")
  public PlatformTransactionManager txManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
  }
 
  // 创建事务管理器2，不需要指定数据库，需要指定 EntityManagerFactory
  @Bean(name = "txManager2")
  public PlatformTransactionManager txManager2(EntityManagerFactory factory) {
      return new JpaTransactionManager(factory);
  }

  // 实现接口 TransactionManagementConfigurer 方法，设置默认的事务管理器
  @Override
  public PlatformTransactionManager annotationDrivenTransactionManager() {
    return txManager2;
  }

  public static void main(String[] args) {
    SpringApplication.run(ProfiledemoApplication.class, args);
  }
}

@Component
public class DevSendMessage implements SendMessage {
  // 使用 value 指定事务管理器
  @Transactional(value="txManager1")
  @Override
  public void send() {
    // ...
  }

  // 使用默认的事务管理器；如果没有默认的事务管理器，就必须制定 value，否则会抛出异常
  @Transactional
  public void send2() {
    // ...
  }
}

// 第二种方式，使用 @TxManager1 注解
// @Target({ElementType.METHOD, ElementType.TYPE})
// @Retention(RetentionPolicy.RUNTIME)
// @Transactional("txManager1")
// public @interface TxManager1 {
// }
```

### how

spring 事务通过 AOP 面向切面编程机制实现，首先从 @Transactional 注解中获取必要的元数据信息，然后构建一个 AOP 代理（spring 事务也支持基于 AspectJ 切面），它使用 TransactionInterceptor 事务拦截器和 PlatformTransactionManager 接口实现类（对接不同数据访问技术的事务操作）来驱动事务。

#### EnableTransactionManagement

@EnableTransactionManagement 注解既会注入默认的 PlatformTransactionManager 实例，又会通过 Spring 的自动机制添加切面（装配 TransactionInterceptor 事务拦截器），自动查询应用上下文中的 @Transactional 注解，并构建 AOP 代理执行事务流程。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
  // 代理模式，基于子类或基于 java 接口
	boolean proxyTargetClass() default false;

  // 切面模式，默认基于代理，可以切换成基于 AspectJ
	AdviceMode mode() default AdviceMode.PROXY;

  // 拦截器顺序
	int order() default Ordered.LOWEST_PRECEDENCE;
}

public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {
  // 通过 AnnotationMetadata 接口获取注解的元数据
  // 由元数据 mode 获取 ProxyTransactionManagementConfiguration 类名等，使用 spring 机制自动装配
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
        // AutoProxyRegistrar 实现 ImportBeanDefinitionRegistrar 接口，会在注册 bean 的时候设置 AOP 代理的方式，基于子类或基于接口
				return new String[] {AutoProxyRegistrar.class.getName(),
        // ProxyTransactionManagementConfiguration 间接实现 ImportAware 接口，可以通过 setImportMetadata 方法获取注解元数据
        // ProxyTransactionManagementConfiguration 属性中包含默认的 PlatformTransactionManager 实例
        // 通过 ProxyTransactionManagementConfiguration 自动加载 TransactionInterceptor 拦截器
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}
}

// 通过 ProxyTransactionManagementConfiguration 自动加载 TransactionInterceptor 拦截器
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {
  // 设置切点、建言等
	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)// 声明 Bean 的分类
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

  // 解析 @Transactional 注解
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

  // 事务拦截器
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}
}
```

#### TransactionInterceptor

TransactionInterceptor 事务拦截器的简要实现原理如下：

![image](transcational.png)

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
	@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
}

public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
	@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// 获取 @transactional 注解属性
    // txAttr 与 TranscationStatus 挂钩，作为 PlatformTransactionManager#getTranscation 的参数，返回 TranscationStatus
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    // 通过 beanFactory 以及 @transactional 注解上的 value 元数据获取 PlatformTransactionManager 实例
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// 构建 TransactionInfo，保存在本地线程中
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// 继续执行拦截器链,当有其他拦截器 match 待执行方法时，则执行该拦截器方法，然后 return
				retVal = invocation.proceedWithInvocation();
			} catch (Throwable ex) {
				// 根据 @transactional 元数据 rollbackOn 进行回滚或调用 PlatformTransactionManager#commit 提交
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			} finally {
				cleanupTransactionInfo(txInfo);
			}
      // 调用 PlatformTransactionManager#commit 提交
			commitTransactionAfterReturning(txInfo);
			return retVal;
		} else {
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
					try {
						return invocation.proceedWithInvocation();
					} catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException) {
								throw (RuntimeException) ex;
							} else {
								throw new ThrowableHolderException(ex);
							}
						} else {
							throwableHolder.throwable = ex;
							return null;
						}
					} finally {
						cleanupTransactionInfo(txInfo);
					}
				});

				if (throwableHolder.throwable != null) {
					throw throwableHolder.throwable;
				}
				return result;
			} catch (ThrowableHolderException ex) {
				throw ex.getCause();
			} catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}
		}
	}
}
```

#### PlatformTransactionManager

PlatformTransactionManager 接口定义了事务执行的策略。不同的数据访问技术有各自的实现，如 JDBC 实现了 DataSourceTranscationManager；JPA 实现了 JPATranscationManager；Hibernate 实现了 HibernateTranscationManager；JDO 实现了 JdoTranscationManager；分布式事务实现了 JtaTranscationManager。在 spring boot 项目中，spring-boot-starter-jdbc 会为应用自动注入 DataSourceTransactionManager 实例；spring-boot-starter-data-jpa 会为应用自动注入 JpaTransactionManager 实例。

```java
public interface PlatformTransactionManager {
  // 如果当前调用堆栈存在匹配事务，就返回新事务或当前的事务
  // TransactionDefinition 定义了事务的元数据，基本与 @Transactional 注解上的元数据相同
  TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

  void commit(TransactionStatus status) throws TransactionException;

  void rollback(TransactionStatus status) throws TransactionException;
}

public interface TransactionStatus extends SavepointManager {
  boolean isNewTransaction();

  boolean hasSavepoint();

  // 调用 TransactionAspectSupport.currentTransactionStatus().setRollbackOnly() 实现回滚
  void setRollbackOnly();

  boolean isRollbackOnly();

  void flush();

  boolean isCompleted();
}
```

#### Transactional

@Transactional 注解元数据包含：

* propagation（事务传播）：默认值为 REQUIRED，事务方法 A 调用事务方法 B，B 将使用与 A 相同的事务，而不创建新事务；当方法 B 发生异常时，整个事务将回滚。当值为 REQUIRED_NEW 时，方法 B 将构建新的事务，异常只导致 B 事务回滚。当值为 NESTED 时，效果与 REQUIRED_NEW 类似，但只支持 JDBC，不支持 JPA、Hibernate。当赋值为 SUPPORTS 时，声明方法调用时有事务就用事务，没有事务就不用事务（比如某方法没有事务，调用它的方法可能有事务）。当赋值为 NOT_SUPPORTS 时，声明方法不在事务中执行，若有事务，在方法调用结束阶段事务都会被挂起。当赋值为 NEVER 时，声明方法不在事务中执行，若有事务则抛出异常。当赋值为 MANDATORY 时，声明方法在事务中执行，若没有事务则抛出异常。
* isolation：指定事务之间的隔离程度。默认值为 DEFAULT，使用当前数据库的默认隔离级别，Oracle、SQL Server 为 READ_COMMITTED，Mysql 为 REPEATABLE_READ。当值为 READ_UNCOMMITTED 时，B 事务可以读取到 A 事务修改但未提交的事务，可能导致脏读、不可重复读、幻读。当值为 READ_COMMITTED 时，B 事务可以读取到 A 事务修改的事务，可能导致不可重复读、幻读，不会导致脏读。当值为 REPEATABLE_READ 时，B 事务可以读取到 A 事务修改的事务，但不可以读取 A 事务已经读取的事务，可能导致不可重复读，不会导致幻读、脏读。当值为 SERIALIZABLE 时，事务顺序执行，不会导致不可重复读、幻读、脏读，但是开销较大。
* timeout：指定事务过期时间，默认为当前数据库的事务过期时间。超过超时时间的事务将会被回滚。
* readOnly：指定当前事务是否为只读事务，即添加 @Transactional 注解的方法只会读取数据库的值，而不会修改数据库的值。
* rollbackFor：指定哪些异常可以引起事务回滚，值为 Throwable 的子类。
* noRollbackFor：指定哪些异常不会引起事务回滚，值为 Throwable 的子类。
* 自定义事务行为。
* 不支持通过远程调用传播事务上下文。

#### TransactionTemplate

同常见的数据访问技术实现了了两种抽象一样（如较低水平的抽象 DataSourceUtils、较高水平的抽象 JDBCTemplate），spring 事务机制既支持使用 @Transactional 注解作声明式编程，又支持使用 TransactionTemplate 或 PlatformTransactionManager 实现类作编程式编程。除了使用 TransactionTemplate 约束事务的行为以外，spring 事务机制也允许使用 TransactionalEventListener 在事务执行的某个阶段定制事件。

```java
public class SimpleService implements Service {
  private final TransactionTemplate transactionTemplate;

  public SimpleService(PlatformTransactionManager transactionManager) {
    this.transactionTemplate = new TransactionTemplate(transactionManager);
    this.transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_UNCOMMITTED);
    this.transactionTemplate.setTimeout(30); // 30 seconds
  }

  public Object someServiceMethod() {
    return transactionTemplate.execute(new TransactionCallbackWithoutResult() {
      protected void doInTransactionWithoutResult(TransactionStatus status) {
        try {
          updateOperation1();
          updateOperation2();
        } catch (SomeBusinessException ex) {
          status.setRollbackOnly();
        }
      }
    });
  }
}
```

### 参考

[spring 事务官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#transaction)
[Java中的事务——全局事务与本地事务](https://blog.csdn.net/w372426096/article/details/78574161)
[Spring 事务 – @Transactional的使用](https://www.jianshu.com/p/befc2d73e487)
[@Transactional原理](https://www.jianshu.com/p/b33c4ff803e0)
[Distributed transactions in Spring, with and without XA](https://www.infoworld.com/article/2077963/distributed-transactions-in-spring--with-and-without-xa.html)
[Java Transaction Design Strategies](https://www.infoq.com/minibooks/JTDS/)