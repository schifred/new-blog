---
title: Servlet
category:
  - 后端
  - java
tags:
  - 后端
  - java
keywords: 'Servlet,java'
abbrlink: 284a2673
date: 2019-03-23 00:00:00
updated: 2019-03-23 00:00:00
---

我们先来看一下，作为 Servlet 容器（也称为 web 容器）的 Tomcat 的一般工作机制：

1. 客户端首次发送请求，Tomcat 将实例化特定的 Servlet 类并执行 init 方法。
2. Tomcat 将请求解析成 request，并将其转发给 servlet。
3. servlet调用 service 方法处理请求，获得 reponse 并发送回客户端。
4. 当客户端再次发送请求，重复 2, 3 步。
5. 当 Servlet 销毁时（卸载应用程序或关闭 Servlet 容器），调用 destory 方法。

Servlet 是运行在 web 服务器上用于接受和处理请求、获得响应的程序。javax.servlet 包提供了一组接口和类，用于定义和描述 Servlet 类和 Servlet 容器（提供 servlet的运行时环境）的契约。

The javax.servlet package contains a number of classes and interfaces that describe and define the contracts between a servlet class and the runtime environment provided for an instance of such a class by a conforming servlet container.

和一般的容器技术相同，Servlet 容器会协调 servlet 的生命周期，并为 servlet 提供全局上下文 ServletContext，以便于使用通用的工具函数、全局配置或缓存、访问全量的 servlet、动态注册 Servlet 等。因此，对于单机环境部署的 web 应用，针对不同的路由会有多个 servlet，servletContext 却只有一个；这也使得在单机环境中 servletContext 可用于缓存全局数据。当然在分布式环境下，每台虚拟机都会有一个 servletContext，不宜再使用 servletContext 缓存数据。

聊回容器技术，抽象类 ServletContext 是由容器实现的，就是说容器在应用启动阶段就会实例化全局上下文，并将 Servlet 类加载到内存中。在 ServletContext 抽象类中，我们可以看见 setInitParameter 方法用于对接全局的初始参数，addServlet 方法用于注册 Servlet（实现类完全可以用实例属性缓存全量的 Servlet）。在 javax.servlet 的设计中，Servlet 的初始参数和路由规则等配置项同 Servlet 呈轻耦合关系，addServlet 方法实际注册的是 servletRegistration，这样就使得 servlet 不能变更路由规则了。

由 servletContext 向上封装构成 servletConfig，该实例既作为 servlet.init 方法的参数，又可以通过 servlet.getServletConfig 方法间接访问。每个 servlet 对应一个 servletConfig。通过这个 servletConfig，既可以获取全局上下文（全局的初始参数可通过上下文进行获取），又在于获取 servlet 自有的初始参数。因此，容器会维护 servletContext, servletConfig 实例，并协调 servlet 生命周期方法的调用。Servlet 包含如下三个生命周期方法：

* init(ServletConfig config) 初始化方法。通常当 Servlet 容器初次接受请求时，就会尝试实例化 Servlet 并调用其 init 方法。如果将 Servlet 配置为 loadOnStartup，那么 init 方法就会在容器启动时调用。
* service(ServletRequest req, ServletResponse res) 处理请求以获取响应。需要注意的是，Servlet 容器会以多进程的方式运行，共享的资源、缓存需要及时作同步。
* destroy() 当 Servlet 移出 Servlet 容器时被调用，如杀死进程等。实现 destory 方法一般用于销毁状态、及时同步资源或缓存。

javax.servlet 开具了 Filter 过滤器的机制，该机制可用于作公共层面的权限校验、日志打印、数据转换等。它同样通过 servletContext 注册和维护，由容器提供 filterConfig 以获取全局上下文及配置。它也具有三个生命周期方法：

* init(FilterConfig filterConfig) 初始化方法。
* doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 过滤逻辑，调用 chain.doFilter 执行下一个过滤器或将请求转交给 servlet。
* destroy() 销毁。

filter 有其适用范围，默认为 DispatcherType.REQUEST 过滤常规请求。当将 Filter 的初始参数 dispatcherTypes 配置为 DispatcherType.FORWARD, DispatcherType.INCLUDE, DispatcherType.ERROR, DispatcherType.ASYNC 时，将分别对 requestDispatcher.forward(req, res) 重定向、 requestDispatcher.include(req, res) 资源包含、404 响应、异步请求作过滤。

### ServletContext

ServletContext 的功能如下：

* 获取资源或查看其媒体类型。
* 注册 servlet 并作管理。
* 注册 filter 并作管理。
* 缓存全局数据，在数据变更时触发 listener。
* 设置 session 超时时间等…

servletContext 及其 attribute 属性均会经由 java.util.EventObject 包装，并由实现类组织事件的触发时机。因此对于 servletContext 的初始化及其 attribute 属性的变动，都可以订制监听器。

```java
// Servlet 容器初始化时调用
public interface ServletContainerInitializer {
    void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException;
}

public interface ServletContext {
    // 全局初始参数
    public String getInitParameter(String name);
    public Enumeration<String> getInitParameterNames();
    public boolean setInitParameter(String name, String value);

    // 获取资源或资源的媒体类型
    public String getMimeType(String file);
    public Set<String> getResourcePaths(String path);
    public URL getResource(String path) throws MalformedURLException;
    public InputStream getResourceAsStream(String path);

    // 获取 requestDispatcher，以重定向或作资源包含
    public RequestDispatcher getRequestDispatcher(String path);
    public RequestDispatcher getNamedDispatcher(String name);

    // 打印日志，日志的名称和类型由容器决定
    public void log(String msg);
    public void log(String message, Throwable throwable);
  
    // 缓存或获取全局状态，其值变更时将会通知 listener
    public Object getAttribute(String name);
    public Enumeration<String> getAttributeNames();
    public void setAttribute(String name, Object object);
    public void removeAttribute(String name);

    // 注册或获取 Servlet
    public ServletRegistration.Dynamic addServlet(String servletName, String className);
    public ServletRegistration.Dynamic addServlet(String servletName, Servlet servlet);
    public ServletRegistration.Dynamic addServlet(String servletName,
            Class<? extends Servlet> servletClass);
    public ServletRegistration.Dynamic addJspFile(String jspName, String jspFile);
    public <T extends Servlet> T createServlet(Class<T> c)
            throws ServletException;
    public ServletRegistration getServletRegistration(String servletName);
    public Map<String, ? extends ServletRegistration> getServletRegistrations();

    // 注册或获取过滤器
    public FilterRegistration.Dynamic addFilter(String filterName, String className);
    public FilterRegistration.Dynamic addFilter(String filterName, Filter filter);
    public FilterRegistration.Dynamic addFilter(String filterName,
            Class<? extends Filter> filterClass);
    public <T extends Filter> T createFilter(Class<T> c) throws ServletException;
    public FilterRegistration getFilterRegistration(String filterName);
    public Map<String, ? extends FilterRegistration> getFilterRegistrations();

    public SessionCookieConfig getSessionCookieConfig();
    public void setSessionTrackingModes(
            Set<SessionTrackingMode> sessionTrackingModes);
    public Set<SessionTrackingMode> getDefaultSessionTrackingModes();
    public Set<SessionTrackingMode> getEffectiveSessionTrackingModes();

    // 添加 listener
    public void addListener(String className);
    public <T extends EventListener> void addListener(T t);
    public void addListener(Class<? extends EventListener> listenerClass);
    public <T extends EventListener> T createListener(Class<T> c)
            throws ServletException;

    // 获取、设置 session 超时时间
    public int getSessionTimeout();
    public void setSessionTimeout(int sessionTimeout);

    // 获取、设置请求体 request body 编码
    public String getRequestCharacterEncoding();
    public void setRequestCharacterEncoding(String encoding);

    // 获取、设置响应体 reponse body 编码
    public String getResponseCharacterEncoding();
    public void setResponseCharacterEncoding(String encoding);

    public JspConfigDescriptor getJspConfigDescriptor();

    public ClassLoader getClassLoader();
}
```

### Servlet

关于 Servlet 的讲解见于上文，我们知道，开发者可以实现特定的 init, service, destory 方法。在讲解 HttpServlet 前，我们先来了解一下 ServletRequest, ServletResponse。

#### req && res

顾名思义，当接受到请求时，Servlet 容器会将其封装为 servletRequest，便于读取请求内容。容器既实现了 ServletRequest 接口，又实现了 ServletInputStream 抽象类，通过 servletInputStream.read 方法可读取请求内容。ServletRequest 接口包含如下功能：

* 访问全局上下文。
* getInputStream 返回 ServletInputStream 实例，便于以二进制形式读取请求体。
* getReader 返回 BufferedReader 实例，便于以字符串形式读取请求体。
* getParameter 获取查询参数。
* getPart, getParts 返回 Part 实例或集合，便于获取 multipart/form-data 媒体类型上传的文件，
* setAttribute 设置缓存数据。
* 获取其他请求相关信息。
* 获取服务器或客户端 ip 地址信息等…

用于处理 http 请求的 HttpServletRequest 接口增加了如下功能：

* getQueryString 获取查询参数。
* getMethod 获取请求方式。
* getRequestURI 获取请求地址。
* getCookies 获取 cookie，返回值为 Cookie 实例构成的数组（Cookie 类用于便捷处理 cookie）。
* getRequestedSessionId 获取 sessionId。
* getSession 获取 session，返回值为 HttpSession 实例（HttpSession 类用于便捷处理 session）。
* getHeader 获取请求头。
* newPushBuilder 通过 PushBuilder 伪造请求等。

servletResponse 用于生成响应。同样，容器既实现了 ServletResponse 接口，又实现了 ServletOutputStream 抽象类，通过 servletOutputStream.write 方法可读取请求内容。servletResponse.getOutputStream 方法返回 ServletOutputStream 实例，便于以二进制形式发送数据；servletResponse.getWriter 方法返回 PrintWriter 实例，便于以字符串形式发送数据。因为 http 会在发送响应前先发送响应头，所以响应头需要提前创建。

继承 ServletResponse 的 HttpServletResponse 接口增加了如下功能：

* 加密 url。
* 添加 cookie。
* 设置响应头。
* 设置状态码。
* sendError 发送错误响应。
* sendRedirect 重定向。

额外的，javax.servlet.http 提供了 HttpServletRequestWrapper, HttpServletResponseWrapper 允许开发者以子类的形式制作适配器，便于快速处理请求和响应。

### HttpServlet

javax.servlet 中的 GenericServlet 抽象类同时实现了 Servlet, ServletConfig 接口，因此 GenericServlet 子类既可以实现 init, service, destory 生命周期方法，又可以获取初始参数、servletConfig。

抽象类 HttpServlet 就是 GenericServlet 的子类，在其保护方法 service 中，将根据请求方式的不同分别调用 doGet 等方法处理请求。HttpServlet 子类可实现的方法包含：

* doGet 处理 get 请求。
* doHead 处理 head 请求，只返回响应头。
* doPost 处理 post 请求。
* doPut 处理 put 请求，用于发送文件等。
* doDelete 处理 delete 请求，用于删除服务端的文件等。
* doOptions 处理 options 请求，通过 ALLOW 响应头获得服务端对请求路由支持的请求方式，比如通过实现 doGet 方式判断是否支持 get 请求。

```java
public abstract class HttpServlet extends GenericServlet {
    private static final String HEADER_IFMODSINCE = "If-Modified-Since";
    protected long getLastModified(HttpServletRequest req) {
        return -1;
    }

    private void maybeSetLastModified(HttpServletResponse resp,
                                      long lastModified) {
        if (resp.containsHeader(HEADER_LASTMOD))
            return;
        if (lastModified >= 0)
            resp.setDateHeader(HEADER_LASTMOD, lastModified);
    }

    protected void service(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

        String method = req.getMethod();

        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                doGet(req, resp);
            } else {
                long ifModifiedSince;
                try {
                    ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                } catch (IllegalArgumentException iae) {
                    ifModifiedSince = -1;
                }
                // 资源更新
                if (ifModifiedSince < (lastModified / 1000 * 1000)) {
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }
        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);
        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);
        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);
        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);
        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req,resp);
        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req,resp);
        } else {
            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);

            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }

    @Override
    public void service(ServletRequest req, ServletResponse res)
        throws ServletException, IOException {

        HttpServletRequest  request;
        HttpServletResponse response;

        try {
            request = (HttpServletRequest) req;
            response = (HttpServletResponse) res;
        } catch (ClassCastException e) {
            throw new ServletException(lStrings.getString("http.non_http"));
        }
        service(request, response);
    }
}
```

### annotation

javax.servlet.annotation 提供了一些常用注解：

* WebServlet 声明 Servlet，设置初始化参数、路由规则等。
* WebFilter 声明 Filter。
* MultipartConfig 声明 Servlet 将用于处理文件上传操作，期望请求头的媒体类型为 multipart/form-data。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MultipartConfig {
    // 容器暂存文件的临时目录
    String location() default "";

    long maxFileSize() default -1L;

    long maxRequestSize() default -1L;

    // 文件多大时写入临时目录，小于此值的作内存缓存
    int fileSizeThreshold() default 0;
}
```

### 并发

Servlet 容器包含可复用线程组成的线程池，避免创建、销毁线程带来的性能消耗。当容器接收到请求时，它将在线程池中取出可用的线程处理请求。当没有可用的线程时，请求将被放入到一个先进先出队列中等待处理。多线程所带来的问题是，Servlet 中的静态属性和实例属性可以被多个线程同时访问，这样就可能会引起一致性问题：在某个线程中已修改的实例属性不能被并行处理的另一个线程感知到。在多线程的情况中，方法创建的对象和变量在发放执行过程中都是安全的，其他线程没法访问到它们。因此，我们需要避免使用静态属性或实例属性缓存可变内容；非到万不得已，可使用 synchronized 同步代码块处理，保证其他线程无法同时执行代码。

### 小结

可借鉴的点：

* 协调组件生命周期的容器，在前端层面可以构建管理 page 的容器。
* 使用抽象类和接口设立规范。

### 参考

[Package javax.servlet](http://tomcat.apache.org/tomcat-8.0-doc/servletapi/javax/servlet/package-summary.html)
[JavaWeb——Servlet（全网最详细教程包括Servlet源码分析）](https://blog.csdn.net/qq_19782019/article/details/80292110)
[ServletContext理解学习](https://blog.csdn.net/tangiwang/article/details/83149693)
[servlet 3.0笔记之servlet的动态注册](http://www.blogjava.net/yongboy/archive/2010/12/30/346209.html)
[Servlet3.0的异步](https://www.cnblogs.com/zr520/p/6103410.html)
[Java Web基础知识之Filter](https://blog.csdn.net/lmy86263/article/details/51219072)
[tomcat 与 nginx，apache的区别及优缺点](https://blog.csdn.net/sky786905664/article/details/79433123)
