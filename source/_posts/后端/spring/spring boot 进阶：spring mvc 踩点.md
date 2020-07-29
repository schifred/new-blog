---
title: spring boot 进阶：spring beans 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 1498e1bd
date: 2020-01-04 06:00:00
updated: 2020-01-04 06:00:00
---

spring mvc 基于 前端控制器模式 设计，通过 DispatcherServlet 这个前端控制器将请求交给可配置的委托组件处理，这就能支持请求处理的灵活性。DispatcherServlet 中请求映射、视图解析、异常处理等功能所使用的组件都通过 spring 机制发现。

### DispatcherServlet 初始化

Servlet3 会主动查询 WebApplicationInitializer 接口的实现类，同时会执行 WebApplicationInitializer 实现类的 onStartup 方法。通过该方法，它会创建 spring 应用上下文并加载 bean（两个应用上下文：一个为 DispatcherServlet 加载控制器、视图解析器等 web 组件；一个加载中间层和数据层等非 web 组件），它同时会创建 DispatcherServlet 实例，且将 DispatcherServlet 实例和 web 过滤器挂载到 ServletContext 上。

![image](dispatchservlet2.png)

#### 应用

以下示例单纯使用 spring + spring webmvc 的场景，没有使用 spring boot。

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
  // 为其中一个 spring 应用上下文（加载数据层组件）指定 RootConfig 配置类或组件
  @Override
  protected Class<?>[] getRootConfigClasses() {
    return null;
  }

  // 为其中一个 spring 应用上下文（加载控制器）指定 WebConfig 配置类或组件
  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] { MyWebConfig.class };
  }

  // 指定 DispatcherServlet 处理的路由
  @Override
  protected String[] getServletMappings() {
    return new String[] { "/" };
  }

  // 指定过滤器
  @Override
  protected Filter[] getServletFilters() {
    return new Filter[] { new HiddenHttpMethodFilter() };
  }
}
```

### @EnableWebMvc

@EnableWebMvc 注解会通过 @Import 加载配置类，配置类又会主动查找 Controller 等 bean，并注册到 HandlerMapping 中，以便在请求触达时使用。

![image](enablewebmvc.png)

#### 应用

配置类可以继承 WebMvcConfigurer，以便配置转换器、拦截器、 cors 映射规则等。以下示例仅展示 cors 规则的配置：

```java
@Configuration
@EnableWebMvc
@ComponentScan("demo.web")// 通过组件扫描加载 Controller 等 web 组件
public class WebConfig implements WebMvcConfigurer {
  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
      .allowedOrigins("https://domain2.com")
      .allowedMethods("PUT", "DELETE")
      .allowedHeaders("header1", "header2", "header3")
      .exposedHeaders("header1", "header2")
      .allowCredentials(true).maxAge(3600);
  }
}

// 附注 RootConfig 配置类
// @Configuration
// @ComponentScan(basePackages={"demo.web"},// 通过组件扫描加载中间层、数据层等非 web 组件
//   excludeFilters={
//     @Filter(type=FilterType.ANNOTATION, value=EnableWebMvc.class)
//   })
// public class RootConfig {
// }
```

### DispatcherServlet 处理请求

DispatcherServlet 作为处理请求的中央调度器。它会使用 HandlerMapping 将请求路由到指定的处理器 handler，HandlerMapping 的默认实现类为 BeanNameUrlHandlerMapping 以及 RequestMappingHandlerMapping。它会使用 HandlerAdapter 在 handlerMethod 之前或之后处理请求或响应，默认实现类 RequestMappingHandlerAdapter 能够解析 Controller 上的注解。ViewResolver 可用于指定视图解析策略；在没有指定视图名的情况下，RequestToViewNameTranslator 用于通过请求获取视图名。MultipartResolver 用于解析文件；LocaleResolver 用于实现国际化。错误处理由 HandlerExceptionResolver 完成，它会将错误转交给 handler 或 view。上述 bean（如添加了 @Controller、@RequestMapping 注解的控制器），DispatcherServlet 均通过 applicationContext#refresh 阶段从已加载的 bean 中获取。

1. 请求触达，DispatcherServlet#doGet 等方法接受到请求。
2. DispatcherServlet#doDispatch 阶段，路由到 Handler（即添加了 @RequestMapping 注解的 Controller）。
3. DispatcherServlet#doDispatch 阶段，通过 HandlerAdapter 处理请求、执行 handler、转化响应。
4. DispatcherServlet#doDispatch 阶段，渲染视图（包含 html、xml、pdf、json 等）或错误。

![image](dispatchservlet.png)

```java
// DispatcherServlet 间接实现了 ApplicationContextAware，可以获得 applicationContext
public class DispatcherServlet extends FrameworkServlet {
  // applicationContext#refresh 阶段从 bean 中获取 HandlerMapping 等 bean
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

  // 其他 doPost、doPut、doDelete 等方法最终都会调用 doDispatch
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				// 从 HandlerMapping 中获取 handler
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // 调用拦截器 interceptor.preHandle 方法进行处理
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// 通过 HandlerAdapter 处理请求、执行 handler、转化响应
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);

        // 调用拦截器 interceptor.postHandle 方法进行处理
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			} catch (Exception ex) {
				dispatchException = ex;
			} catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}

      // 渲染视图或错误
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
	}
}
```

### @RequestMapping

RequestMappingHandlerAdapter 实现了 BeanFactoryAware, InitializingBean 接口。依循 spring bean 的生命周期，RequestMappingHandlerAdapter 首先会找到添加了 @ControllerAdvice 注解的类，并将该类中添加了 @RequestMapping、@ModelAttribute、@InitBinder 等注解的方法存入指定的 cache 中；加载参数解析器、绑定数据解析器、返回值处理器。等到请求触达时，它会按照参数解析器、handlerMethod、返回值处理器的方式处理请求和响应，最终响应以 ModelAndView 实例的形式对外输出。

![image](requestmapping.png)

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
      // 从 cache 中获取 @InitBinder 注解的方法以及数据绑定解析器，构建 WebDataBinderFactory 实例
      // binderFactory 既在 InvocableHandlerMethod#getMethodArgumentValues 阶段处理参数
      // 又在 ModelFactory#updateModel 阶段处理响应
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      
      // 从 cache 中获取 @RequestMapping、@ModelAttribute 注解的方法，构建 ModelFactory 实例
      // binderFactory 既在 InvocableHandlerMethod#getMethodArgumentValues 阶段处理参数
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
      // 添加参数解析器
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
      // 添加返回值处理器
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
      // 添加数据绑定工厂类
			invocableMethod.setDataBinderFactory(binderFactory);
      // 添加参数名辨识器，用于判断参数是否需要经由 binderFactory 处理等
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

      // 管理请求处理过程中所使用到的数据
			ModelAndViewContainer mavContainer = new ModelAndViewContainer();

      // 在重定向之前从请求返回只读“输入”闪存属性
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
      // 按照以下的顺序填充 mavContainer#defaultModel（ModelMap实例）中
      // a. 填充 @SessionAttributes 注解的可使用的会话属性
      // b. 通过执行 @ModelAttribute 注解的方法填充数据
      // c. 从 @ModelAttribute 注解的参数上找到 @SessionAttributes 指定属性并填充
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
      // 是否在重定向时忽略默认，默认 false
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

      // 处理异步请求
			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

      // 如果有并发结果
			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

      // 依次调用参数解析器、handlerMethod、返回值处理
			invocableMethod.invokeAndHandle(webRequest, mavContainer);

			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

      // 使用数据绑定解析器处理响应，响应内容有二：视图 view；视图中使用的模型 model（即 mavContainer#defaultModel）
			return getModelAndView(mavContainer, modelFactory, webRequest);
		} finally {
			webRequest.requestCompleted();
		}
	}
}
```

### 参数解析器

参数解析器均需实现 HandlerMethodArgumentResolver 接口，该接口约定了两个方法：supportsParameter 是否解析器可处理的参数；resolveArgument 解析参数。AbstractNamedValueMethodArgumentResolver 是其抽象实现类。@PathVariable、@MatrixVariable、@RequestParam、@RequestHeader、@CookieValue、@RequestBody、@RequestPart、@ModelAttribute、@SessionAttribute、@SessionAttributes、@RequestAttribute 等注解均基于这个抽象类上解析实现。


```java
public abstract class AbstractNamedValueMethodArgumentResolver implements HandlerMethodArgumentResolver {
  @Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    // RequestParam(name = "a", required = true, defaultValue = "3") => NamedValueInfo{name="a",required=true,defaultValue="3"}
		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		MethodParameter nestedParameter = parameter.nestedIfOptional();

    // 处理 name，因为它可能是表达式或占位符
		Object resolvedName = resolveStringValue(namedValueInfo.name);
		if (resolvedName == null) {
			throw new IllegalArgumentException(
					"Specified name must not resolve to null: [" + namedValueInfo.name + "]");
		}

    // 从 webRequest 装填实际的参数值，由子类实现
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveStringValue(namedValueInfo.defaultValue);
			} else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveStringValue(namedValueInfo.defaultValue);
		}

    // 通过数据绑定器转换值
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			} catch (ConversionNotSupportedException ex) {
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			} catch (TypeMismatchException ex) {
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());

			}
		}

    // 调用子类钩子
		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

		return arg;
	}
}
```

#### 应用

```java
@Controller
@RequestMapping("/spittles")
public class SpittleController {
  @Resource
  SpittleRepository spittleRepository;

  @RequestMapping(value="/{spittleId}", method=RequestMethod.GET)
  public String spittles(@PathVariable long spittleId, Model model){
    Spittle spittle = spittleRepository.findOne(spittleId);

    if (spittle == null){// 由 spring mvc 为 404 异常添加 http 状态码
      throw new SpittleNotFoundException();
    }

    // 为 spittle 视图传入模型数据
    model.addAttributes(spittle);
    return "spittle";
  }
}

@ResponseStatus(value=HttpStatus.NOT_FOUND, reason="Spittle Not Found")
public class SpittleNotFoundException extends RuntimeException {
}

@Test
public void shouldShowSpittle() throws Exception {
  SpittleController controller = new SpittleController();
  MockMvc mockMvc = standaloneSetup(controller).build();

  mockMvc.perform(get("/spittles/10086"))
    .addExcept(view().name("spittle"));
}
```

### 参考

[spring 官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc)
[理解SpringMvc架构以及流程](https://www.jianshu.com/p/6433a07f909c)
[Spring MVC源码分析（四）：SpringMVC的HandlerMapping和HandlerAdapter的体系结构设计与实现](https://blog.csdn.net/u010013573/article/details/86550326)
[Spring Web MVC（一）|前端控制器-DispatcherServlet](https://blog.csdn.net/qq_32328959/article/details/90247411)
[SpringMVC之分析RequestMappingHandlerAdapter（一）](https://blog.csdn.net/zknxx/article/details/78336199)
[流程图解Spring Framework（十三）Spring MVC RequestMappingHandlerAdapter 是如何处理@RequestMapping的？](https://blog.csdn.net/u013076044/article/details/88086156)