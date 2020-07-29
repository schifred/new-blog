---
title: spring boot 进阶：spring boot 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: df94d6d
date: 2019-12-14 06:00:00
updated: 2019-12-14 06:00:00
---

### why & what

介于 spring 应用配置较为复杂，spring boot 就应运而生了，其目的即在于简化 spring 项目中依赖的配置流程。因此 spring boot 集成了 spring 的以下能力，或者通过定制 starter 的方式简化了以下能力的配置形式：

* Web Applications：spring boot 内嵌 Tomcat、Jetty、Undertow、Netty 服务器，便于一键启动；支持使用 spring-boot-starter-webflux 模块启动响应式 web 应用。借助于 spring mvc，spring boot 将自动注册 Converter，用于转换请求和响应；支持通过 JsonSerializer、JsonDeserializer 转化 json 数据；支持使用 FreeMarker、Groovy、Thymeleaf、Mustache 等模板引擎；支持通过 WebMvcConfigurer 定制静态资源位、提供 cors 跨域能力等。可参考 [文档](https://docs.spring.io/spring-boot/docs/2.2.1.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc)。
* SQL：提供使用 SQL 数据库的广泛支持，从 JDBC 到对象关系映射框架（如 Hibernate、Mybatis）等。
* NoSQL：支持 Redis、MongoDB、Neo4j、Elasticssearch、Solr Cassandra、Couchbase、LDAP 等 NoSQL 数据库。
* Caching：允许在方法上使用 @Cachable，该注解将根据请求参数作缓存。默认使用内存缓存，也支持 Generic、JCache、EhCache、Hazelcast、Infinispan、Couchbase、Redis、Caffeine、Simple 等缓存服务（spring boot 会自动检测）。
* Hazelcast：支持 Hazelcast 基于内存的数据存储方案。
* Quartz Scheduler：支持通过引入 spring-boot-starter-quartz 启用定时器任务，详情可参考 [Quartz 定时任务（Scheduler）的 3 种实现方式](https://blog.csdn.net/jiangyu1013/article/details/81111898)。
* Task Execution and Scheduling：在 Executor 缺席的情况下，spring boot 会自动装配 ThreadPoolTaskExecutor 用于处理异步任务（通过 @EnableAsync 注解使用）或异步请求，线程池的使用也可以通过配置文件约束。可参考 Quartz 定时任务（Scheduler）的 3 种实现方式。
* Message：支持 JMS、RabbitMQ、Kafka、WebSocket 等消息系统。
* Web Services：自动装配 WebServiceTemplateBuilder，便于创建 WebServiceTemplate 实例以调用远程 web 服务。
* Rest Services：自动装配 RestTemplateBuilder，凭此可以创建 RestTemplate 实例，然后调用 rest 服务；同样的，WebClientBuilder 可用于创建 WebClient 实例，然后调用 rest 服务。
* WebSockets：会为内置的 Tomcat、Jetty、Undertow 容器自动装配 WebSockets，也允许通过 spring-boot-starter-webflux 模块为响应式应用提供 WebSocket 编程能力。
* Validation：借助于 bean 校验机制 —— JSR-303 规范的实现（如 Hibernate validator），spring boot 允许使用 @Validated 对 bean 作校验。
* Distributed Transactions：通过使用 Atomikos 或 Bitronix 嵌入式事务管理器，spring boot 支持 JTA 分布式事务编程规范。当探知到 JTA 环境时，JtaTransactionManager 接口就会用于管理事务，JMS、DataSource、JPA 等 bean 也会被自动装配。编码时使用 @Transactional 就可以实现事务管理。可参考 [Spring的全局事务JTA](https://www.jianshu.com/p/3938e7172443)。
* 日志：支持 Commons Logging、Java Util Logging、Log4J2、Logback 等日志服务。
* 测试：支持通过 spring-boot-starter-test 测试应用。spring-boot-starter-test 包含 JUnit 5、Spring Test & Spring Boot Test、AssertJ、Hamcrest、Mockito、JSONassert、JsonPath。
* 监控：更好地掌握应用的运行状态。

以上能力的大集成或者配置的简化，均借助于 spring boot 的如下特性：

* SpringApplication：通过劫持 spring 应用启动过程的方式，集中管理配置项。
* Externalized Configuration：允许通过 properties、yaml 文件或环境变量、命令行参数等外部配置启动 spring 应用，并能通过 spring.profiles.active 区分不同环境。这些配置项也能通过 @Value、* @ConfigurationProperties 绑定到 bean 或结构化对象中。外部配置抽象为 PropertySource，其优先级大体为命令行参数、环境变量、指定 profile 环境的 properties 或 yaml 文件、不含 profile 环境的 properties 或 yaml 文件。
* Auto Configuration：允许通过自动配置的定制 starter，用以简化配置流程、自动注入 bean 等。

### sample

#### Auto-configuration

通过自动装配技术开发一个 starter，用于在控制台打印每次访问的 URI。示例来自 《Spring Cloud 微服务架构进阶》。

```java
// 定义过滤器
public class LogFilter implements Filter {
	private Logger logger = LoggerFactory.getLogger(LogFilter.class);
	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		logger.info("logFilter init...");
	}
	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, 
	  FilterChain filterChain) throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) servletRequest;
		logger.info("uri () is working.", request.getRequestURI());
		filterChain.doFilter(servletRequest, servletResponse);
	}
	@Override
	public void destory() {
		logger.info("logFilter destory...");
	}
}

// 将 LogFilter 封装成 spring bean
public class LogFilterRegistrationBean extends FilterRegistrationBean<LogFilter>{
	public LogFilterRegistrationBean(){
		super();
		this.setFilter(new LogFilter());
		this.addUrlPatterns("/*");
		this.setName("LogFilter");
		this.setOrder(1);
	}
}

// 定义自动配置类
@Configuration
@ConditionalOnClass({LogFilterRegistrationBean.class, LogFilter.class})
public class LogFilterAutoConfiguration {
	@Bean
	@ConditionalOnMissingBean(LogFilterRegistrationBean.class)
	public LogFilterAutoConfiguration logFilterAutoConfiguration() {
		return new LogFilterRegistrationBean();
	}
}

// 方法 1. 通过注解将配置类引入 spring 扫描范围内，在工程中只要对启动类使用 @EnableLogFilter 注解即可
@Target({ElementType.class})
@Retention(RetentionPolicy.RUNTIME)
@Import(LogFilterAutoConfiguration.class)// 引入自动配置类
public @interface EnableLogFilter {
}

// 方法 2. 通过解析 resource/META-INF/spring.factories 加载自动配置类，在工程中同样使用 @EnableLogFilter 注解
// 		org.springframework.boot.autoconfigure.EnableLogFilter=\
// 		com.mycorp.libx.autoconfigure.LogFilterAutoConfiguration
public class EnableLogFilterImportSelector implements DeferredImportSelector, 
	BeanClassLoaderAware, EnvironmentAware {
	private static final Logger logger = LoggerFactor.getLogger(EnableLogFilterImportSelector.class);
	private Class annotationClass = EnableLogFilter.class;
	@Override
	public String[] selectImports(AnnotationMetadata metadata) {// 由 @EnableLogFilter 注解名决定解析属性
		if (!isEnabled()) {
			return new String[0];
		}
		AnnotationAttributes attributes = AnnotationAttributes.fromMap(
			metadata.getAnnotationAttributes(this.annotationClass.getName(), true)
		);
		Assert.notNull(attributes, "can not be null...");
		List<String> factories = new ArrayList<>(new LinkedHashSet<>(
			SpringFactoriesLoader.loadFactoryNames(this.annotationClass, this.beanClassLoader)
		));
		if (factories.isEmpty() && !hasDefaultFactory()) {
			throw new IllegalStateException("...");
		}
		if (factories.size() > 1) {
			logger.warn("More than one implemetion");
		}
		return factories.toArray(new String(factories.size()));
	}
}
@Target({ElementType.class})
@Retention(RetentionPolicy.RUNTIME)
@Import(EnableLogFilterImportSelector.class)// 引入自动配置类
public @interface EnableLogFilter {// 同步修改 EnableLogFilter
}
```

### how

#### SpringApplication

为了简化 spring 的配置流程，spring boot 通过 SpringApplication 劫持了 spring 的启动过程（通过显式调用 SpringApplication#run 方法启动）。在启动过程中，可定制的 FailureAnalyzer 将分析启动失败的原因并打印在日志上。如果没有错误提示日志，可以尝试 debug 模式启动。SpringApplication 包含但不限于如下功能点：

* 在 spring 机制的基础上，spring boot 支持 bean 的懒初始化（即在请求到达时才创建 bean），这样可以节省应用启动的时间。缺点是没法在启动过程中发现 bean 的配置问题，也没法保证运行时创建的 bean 不会造成 jvm 内存不足。懒初始化默认禁用，可以使用 java 编码或 lazy-initialization 配置项启用。
* 允许通过 SpringApplicationBuilder 创建多个 ApplicationContext 实例（构成父子结构的层级关系），环境变量会在 ApplicationContext 实例中共享，web 组件只能运行在子 ApplicationContext 实例上。
* spring boot 会根据应用类型是否为 servlet、webflux 或上述两者之外选用不同的 ApplicationContext 实现类。
* 通过 spring 事件机制，SpringApplication#addListeners 方法或 META-INF/spring.factories 配置可用于添加启动时的事件监听器。子 ApplicationContext 实例的事件会冒泡到父级，需要区分 ApplicationContext。一般不使用事件。
* 允许使用 ApplicationArguments 接口访问启动参数 args。

以下源码仅简要地展示 SpringApplication 的执行逻辑，其深入部分另作专题分析：

```java
public class SpringApplication {
  // 首参默认为 null，次参为应用的启动类
  public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));

    // 判断应用类型：基于 spring mvc 的 servlet 应用、基于 webflux 的响应式应用、以上两种之外
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		// 加载 ApplicationContextInitializer，使用类加载机制加载 META-INF/spring.factories 配置中的类
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		// 加载 ApplicationListener
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		// 通过 RuntimeException 执行堆栈获取到 ApplicationClass 
		this.mainApplicationClass = deduceMainApplicationClass();
	}

  public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();// 计算执行时间
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			// environment 实例可用于获取 PropertySources 配置信息、Profiles 环境信息
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);// 打印 banner
			// 根据 webApplicationType 应用类型选用 ApplicationContext 实现类并创建实例
			// spring 机制：ApplicationContext 实例中的 getBeanFactory 可用于获取 BeanFactory 实例，以便注册 bean
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			// 调用 ApplicationContext 实例的 refresh 方法
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
}

public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
}
```

#### Auto-configuration

自动配置技术通常会用于制作公司内部的公共库或开源库，即如上文 smaple 一节中的制作 starter 部分。配置类通过 @Configuration 注解声明；@ConditionalOnClass、@ConditionalOnMissingBean 等注解声明配置类加载的条件。@ConditionalOnClass 在某个类存在于类路径时予以自动配置；@ConditionalOnMissingBean 在某个 bean （如用户配置类）不存在于 spring 上下文中时予以自动配置。@AutoConfigureAfter、@AutoConfigureBefore、@AutoConfigureOrder、@Order 可用于约束配置类加载的顺序。

自动配置类的加载与否取决于用户配置或其他条件，因此测试自动配置类就会变得困难。因此 spring boot 提供了对自动配置类的测试方法，即通过 ApplicationContextRunner 予以测试。详情可参考 [文档](https://docs.spring.io/spring-boot/docs/2.2.1.RELEASE/reference/html/spring-boot-features.html#boot-features-test-autoconfig)。

配置类有两种，spring-boot 内置的配置类、开源库等提供的配置类。对于开源库，上文已经表明，基于 @import 实现 @EnableXxx 注解，就可以自动加载配置类（对于 jar 包，通过自动扫描的方式加载 bean 是无效的，需要使用 @Import 注解去加载这些 bean）。对于内置配置类，spring-boot-autoconfigure 包提供了 @SpringBootApplication 注解，它用于加在启动类上。该注解基于 @EnableAutoConfiguration 注解实现，其能力就是自动加载 META-INF/spring.factories 文件中声明的配置类：

![image](spring.factories.png)

```java
// 通过 @Import、AutoConfigurationImportSelector 自动装配 META-INF/spring.factories 中的配置类
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}

class ImportAutoConfigurationImportSelector extends AutoConfigurationImportSelector
		implements DeterminableImports {
}

public class AutoConfigurationImportSelector
		implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
		BeanFactoryAware, EnvironmentAware, Ordered {
	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		// 通过 ClassLoader 获取 META-INF/spring-autoconfigure-metadata.properties 资源的 URL，然后解析获取配置信息（默认配置）
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		// 加载 META-INF/spring.factories 配置文件中的自动配置类（即上文中的 LogFilterAutoConfiguration 自动配置类）
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(
				autoConfigurationMetadata, annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
}
```

### 后记

解读 java 技术属于自不量力，因此文章题名“我看 XXX”，用于表明这些文章既是个性化解读的，又会存在很多错谬。

### 参考

[spring boot 官方文档](https://spring.io/projects/spring-boot/#overview)
[spring boot 启动原理](https://www.jianshu.com/p/ef6f0c0de38f)
[SpringBoot源码研究之Start](https://blog.csdn.net/lqzkcx3/article/details/78405229)
[Webflux 核心](https://blog.csdn.net/liyantianmin/article/details/91390090)
[深究Spring中Bean的生命周期](https://www.cnblogs.com/javazhiyin/p/10905294.html)