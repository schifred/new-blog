---
title: spring boot 进阶：spring beans 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 5f8c5aae
date: 2020-01-12 06:00:00
updated: 2020-01-12 06:00:00
---

先介绍两个重要的概念（详情可以参看 [BeanFactory 和 FactoryBean 的区别](https://www.cnblogs.com/xingzc/p/9138256.html)）：

* BeanFactory：作为接口，定义了 Spring IOC 容器最底层的编程规范，职能包含实例化、定位、配置应用程序中的 bean 及建立 bean 之间的依赖。
* FactoryBean：用于实例化 bean。Spring 有两种 bean：通过反射机制使用 class 创建的 bean，如添加了 @Component 注解的 bean；通过实现了 FactoryBean 接口的类创建 bean，在 FactoryBean#getObject 会创建所需要的类实例。

在 spring 中，应用上下文 ApplicationContext 在 refresh 方法执行期间，会创建 BeanFactory 实例并注册 BeanDefiniton（定义了 bean 的基础属性：scope 是否单例、lazyInit 是否懒加载等）；然后通过调用 BeanFactory#getBean 方法可以创建 bean。ApplicationContext#getBeanFactory 可用于获取 BeanFactory 实例；ApplicationContext#getBean 可用于获取 bean。

![image](bean.png)

### ApplicationContext

作为应用上下文，ApplicationContext#refresh 执行过程既会创建 BeanFactory、注册 BeanDefiniton，又集成了国际化、事件广播机制。该接口有以下实现类：

* AnnotationConfigApplicationContext：从一或多个配置类中加载上下文定义，适用于 java 注解方式加载 bean
* ClassPathXmlApplicationContext：从类路径下的一或多个 xml 配置文件中加载上下文定义，适用于 xml 配置方式
* FileSystemXmlApplicationContext：从文件系统下的一或多个 xml 配置文件中加载上下文定义
* AnnotationConfigWebApplicationContext：专门为 web 应用准备的，适用于注解方式
* XmlWebApplicationContext：从 web 应用下的一或多个xml配置文件加载上下文定义，适用于 xml 配置方式

以下是 AnnotationConfigApplicationContext 的使用情形：

```java
@Configuration
public class ManConfig {
	@Bean
	public Man man() {
			return new Man();
	}
}

public class Test {
	public static void main(String[] args) {
		// 调用 ApplicationContext#refresh 方法，注册 BeanDefiniton
		ApplicationContext context = new AnnotationConfigApplicationContext(ManConfig.class);
		// 获取 bean
		Man man = context.getBean(Man.class);
		man.driveCar();
	}
}
```

实际上，在 AnnotationConfigApplicationContext 实例化过程中，即会调用 refresh 方法。对于非懒加载的 bean，refresh 方法执行期间即会予以加载。当然，在 AnnotationConfigApplicationContext 实例化过程中，它会扫描 basePackages 以便筛选出添加了 @Component 注解的类。refresh 源码见下：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 设置环境变量、容器开关和激活标志等
			prepareRefresh();

      // 调用子类实现的 refreshBeanFactory 方法创建 BeanFactory 实例，并加载 BeanDefiniton
      // BeanFactory 实例如 AbstractRefreshableApplicationContext 中的 DefaultListableBeanFactory 实例
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 配置 BeanFactory 所用的类加载器、bean 表达式解析器、属性编辑器等
			prepareBeanFactory(beanFactory);

			try {
				// 执行子类的 postProcessBeanFactory，可用于（通过应用的配置文件）修改 BeanDefiniton
				postProcessBeanFactory(beanFactory);

				// 执行 BeanFactoryPostProcessor#postProcessBeanFactory，可用于修改 BeanDefiniton
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册 bean 初始化钩子 BeanPostProcessor
				registerBeanPostProcessors(beanFactory);

				// 加载 MessageSource 这个特殊的 bean，用于国际化处理
				initMessageSource();

				// 初始化 ApplicationEventMulticaster 这个特殊的 bean，用于事件广播
				initApplicationEventMulticaster();

				// 执行子类的 onRefresh 方法，初始化特殊的 bean
				onRefresh();

				// 加载 ApplicationListener（特殊的 bean 或通过上下文加载），执行 earlyApplicationEvents 前置事件
				registerListeners();

				// 加载 ConversionService 这个特殊的 bean
				// BeanFactory 添加 EmbeddedValueResolver，用于解析配置文件的属性
				// 加载 LoadTimeWeaverAware 这个特殊的 bean，允许织入第三方模块，如 AspectJ
				// 将 BeanFactory 中的 TempClassLoader 置为 null，终止其工作
				// 执行 BeanFactory#freezeConfiguration，冻结 BeanDefiniton
				// 执行 BeanFactory#preInstantiateSingletons，调用 getBean 方法提前加载无需懒加载的 bean
				finishBeanFactoryInitialization(beanFactory);

				// 完成刷新过程，发布应用事件
				finishRefresh();
			} catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				destroyBeans();

				// 将容器激活标识置为否值
				cancelRefresh(ex);

				throw ex;
			} finally {
				// 重置缓存，因为 bean 所用及的元数据将不必使用
				resetCommonCaches();
			}
		}
	}
}
```

### BeanFactory

BeanFactory 接口的默认实现类的 DefaultListableBeanFactory。以下是相关类图：

![image](beanfactory.png)

AbstractBeanFactory 类是 BeanFactory 接口的抽象实现类，在它的 getBean 方法执行期间，如果 bean 还没创建，它就会创建这个 bean；如果已经创建了这个 bean，并且这个 bean 是单例，spring 就会从缓存中获取这个 bean。

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}

	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		final String beanName = transformedBeanName(name);
		Object bean;

		// 获取缓存的单例 bean
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				} else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}

			// （完成 FactoryBean 相关处理，）获取 bean 实例
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);

		} else {
			// 单例正在（循环引用）创建过程中，报错
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 当 IOC 容器 BeanFactory 中不存在 BeanDefinition，向上查找祖先容器
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				} else if (args != null) {
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				} else if (requiredType != null) {
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				} else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				// 合并祖先容器的 BeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 获取（或创建）依赖 bean
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 创建单例 bean
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				} else if (mbd.isPrototype()) {
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				} else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			} catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 类型转换
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			} catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
}
```

### 参考

[Spring 源码解析：高级容器的扩展内幕](https://www.codercto.com/a/3624.html)
[SpringBean 工作原理详解](https://www.cnblogs.com/yoci/p/10642553.html)
[Spring-BeanFactory基本工作流程](https://www.cnblogs.com/Jolivan/p/9168108.html)
[Spring 创建 bean 机制](https://www.cnblogs.com/Jolivan/p/9226289.html)
[spring容器之创建bean实例](https://www.jianshu.com/p/39edfa250e4e)
[使用 BeanPostProcessor 制作 ab 脚本的 bean](https://blog.csdn.net/zwjyyy1203/article/details/89576084)
[使用BeanFactoryPostProcessor——这种姿势不要用](https://www.jianshu.com/p/3d099ea43b0e)
[postProcessBeanFactory方法分析](https://blog.csdn.net/cgj296645438/article/details/80119319)
[Spring源码分析-MessageSource](https://blog.csdn.net/sid1109217623/article/details/84065725)
[ApplicationEventMulticaster 事件广播](https://www.cnblogs.com/jyyzzjl/p/5476546.html)
[Spring 事件机制初始化流程](https://www.jianshu.com/p/151e0b67c5b6)
[Spring源码finishBeanFactoryInitialization和getBean](https://www.jianshu.com/p/9d8e4fb5b162)
[Spring数据转换–ConversionService](https://www.cnblogs.com/arax/p/8551720.html)
[AOP静态代理-代码织入](https://www.cnblogs.com/wade-luffy/p/6078446.html)
[DetaultListableBeanFactory分层关系解析一](https://www.jianshu.com/p/7e845db823cf)