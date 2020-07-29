---
title: spring boot 进阶：spring security 踩点
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 5c27ae9f
date: 2019-12-01 06:00:00
updated: 2019-12-01 06:00:00
---

### 认证和授权

认证即校验用户的身份。认证的方式有：

* 使用回话 ID 作匿名认证，此时用户不需要登录，使用场景如论坛中匿名访问帖子；
* 密码认证（对所有用户，密码都是相同的）；用户名 + 密码认证；
* 使用 WWW-Authenticate、Authenticate 作基本认证。当服务器接收到一个缺少凭证请求，返回 WWW-Authenticate 响应头；接收响应的浏览器就会弹出模拟窗口，促使用户输入用户名和密码；用户名和密码又会以 base64 编码，并作为 Authenticate 请求头的内容，发送到服务器；等认证成功后，服务器就会返回一个正常的响应。基本认证允许用户将凭证内嵌在 URL 或命令行客户端中，形式如 Wget 或 cURL，具体的值如 http://username:password@www.example.com，那么凭证会自动转换成 Authenticate 头。基本认证可以使用 https 协议提高安全性。
* 摘要认证。首先客户端发起请求，服务端返回 WWW-Authenticate 响应头中会包含 algorithm（创建哈希值的算法，”MD5” 或 “MD5-sess”）、qop（保护的质量，”auth” 或 “auth-int” 或 “auth,auth-int）、opaque（随机字符串数据，作完整性校验）、nonce（服务器随机数，在两个 401 响应之间永远不会重复，客户端需要发送相同的随机数）。然后客户端再次发送请求，Authenticate 请求头中会包含 algorithm、qop、opaque、nonce 以及 username、uri、nc、cnonce（期望值，每次请求都会重新生成，且不能与 nc 重复）、response。当服务端接受到请求时，将重新计算所有的哈希值，如果最后的哈希值能匹配 response 参数中的值时，认证就通过了。nc、cnonce 在每次发送的请求均会不同，这样可以阻止重放，避免被黑客获知信息。
* windows 认证，即使用 windows 域控制器进行验证。IE、Chrome 支持 windows 验证。这一技术已过时，这里列举的目的在于可以使用终端凭证验证用户。
* 客户端证书认证，最安全的认证协议之一。首先在注册（通常是创建用户名和密码）时，服务器将告知浏览器生成公钥和私钥，浏览器存储私钥，将公钥发送给服务器。自此之后，公钥可以作为服务器识别浏览器的标识，且浏览器会使用私钥对通信进行签名。服务器接收到 https 请求后，将使用服务器 SSL 证书标识自身，并将自己的公钥和使用私钥签名的数据发送给浏览器。
* 智能卡，就是一个特殊的集成电路，如 ID 卡或磁性员工卡。
* 生物识别，就是使用指纹、声纹、虹膜扫描、DNA 或用户的其他生物身份去认证用户。
* 基于声明的认证，如 OAuth、SAML 协议都基于声明的认证机制。用户在访问应用时，应用将重定向到 Auth URL，然后用户携带凭证访问第三方认证提供者（如登录），第三方提供者将提供令牌给用户；用户再携带令牌访问应用，应用会将令牌发送给第三方认证机构作验证，通过认证后便能获取到应用上的资源。基于声明的认证将在下文详解。
* 多因素认证，即在以上认证的基础上，添加随机数验证（如手机校验码）等手段。

权限模型通常基于活动（用户可操作行为）、角色（用户可操作行为集）、用户组（某类用户可操作行为），阿里的 ACL 权限模型即基于此设计。详细内容另作专题剖析。java.security.Principal 接口可用于实现鉴权操作。当用户通过认证后，程式将用户身份标识存入 Principal 中，应用代码就可以通过检查 Principal（存在于安全上下文中）包含的标识来判断用户是否通过了认证。Principal 也可以保存用户被授权操作的活动，应用代码就可以通过检查 Principal 获得用户的可执行行为。基于声明的授权指的是由第三方认证用户的身份，再由应用识别用户的权限。

### Spring Security

Spring Security 提供了认证和授权服务，也可以防止 csrf 攻击，并提供 jsp 或 Thymeleaf 模板展示用户授权内容等。它基于 spring AOP 以及 servlet 规范中的 filter 机制实现，能够在 Web 请求和方法调用级别处理身份认证和授权。说到底，Spring Security 处理 web 请求的过滤器仍是基于 spring 框架实现的，即通过 AbstractSecurityWebApplicationInitializer 实现 WebApplicationInitializer 接口，并在 onStartup 方法执行过程中使用 servletContext.addFilter 挂载过滤器，内置过滤器是使用 DelegatingFilterProxy 挂载的 SecurityFilterChain，允许用户添加定制的过滤器。

![image](spring-security-filter.png)

以下是 spring security 加载和配置过滤器的主要实现源码：

```java
// 利用 spring 机制将 SpringSecurityFilterChain 等过滤器注册到 servlet 中
public abstract class AbstractSecurityWebApplicationInitializer implements WebApplicationInitializer {
    public final void onStartup(ServletContext servletContext) throws ServletException {
        this.beforeSpringSecurityFilterChain(servletContext);
        if (this.configurationClasses != null) {
            AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
            rootAppContext.register(this.configurationClasses);
            servletContext.addListener(new ContextLoaderListener(rootAppContext));
        }

        if (this.enableHttpSessionEventPublisher()) {
            servletContext.addListener("org.springframework.security.web.session.HttpSessionEventPublisher");
        }

        servletContext.setSessionTrackingModes(this.getSessionTrackingModes());
        this.insertSpringSecurityFilterChain(servletContext);
        this.afterSpringSecurityFilterChain(servletContext);
    }

    private void insertSpringSecurityFilterChain(ServletContext servletContext) {
        String filterName = "springSecurityFilterChain";
        DelegatingFilterProxy springSecurityFilterChain = new DelegatingFilterProxy(filterName);
        String contextAttribute = this.getWebApplicationContextAttribute();
        if (contextAttribute != null) {
            springSecurityFilterChain.setContextAttribute(contextAttribute);
        }

        this.registerFilter(servletContext, true, filterName, springSecurityFilterChain);
    }

    private final void registerFilter(ServletContext servletContext, boolean insertBeforeOtherFilters, String filterName, Filter filter) {
        Dynamic registration = servletContext.addFilter(filterName, filter);
        if (registration == null) {
            throw new IllegalStateException("Duplicate Filter registration for '" + filterName + "'. Check to ensure the Filter is only configured once.");
        } else {
            registration.setAsyncSupported(this.isAsyncSecuritySupported());
            EnumSet<DispatcherType> dispatcherTypes = this.getSecurityDispatcherTypes();
            registration.addMappingForUrlPatterns(dispatcherTypes, !insertBeforeOtherFilters, new String[]{"/*"});
        }
    }
}

// 通过 @Configuration 为 spring-security 添加过滤器、权限规则等
@Order(100)
public abstract class WebSecurityConfigurerAdapter implements WebSecurityConfigurer<WebSecurity> {
    public void init(final WebSecurity web) throws Exception {
        final HttpSecurity http = this.getHttp();
        web.addSecurityFilterChainBuilder(http).postBuildAction(new Runnable() {
            public void run() {
                FilterSecurityInterceptor securityInterceptor = (FilterSecurityInterceptor)http.getSharedObject(FilterSecurityInterceptor.class);
                web.securityInterceptor(securityInterceptor);
            }
        });
    }

    protected final HttpSecurity getHttp() throws Exception {
        if (this.http != null) {
            return this.http;
        } else {
            DefaultAuthenticationEventPublisher eventPublisher = (DefaultAuthenticationEventPublisher)this.objectPostProcessor.postProcess(new DefaultAuthenticationEventPublisher());
            this.localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);
            AuthenticationManager authenticationManager = this.authenticationManager();
            this.authenticationBuilder.parentAuthenticationManager(authenticationManager);
            this.authenticationBuilder.authenticationEventPublisher(eventPublisher);
            Map<Class<? extends Object>, Object> sharedObjects = this.createSharedObjects();
            this.http = new HttpSecurity(this.objectPostProcessor, this.authenticationBuilder, sharedObjects);
            if (!this.disableDefaults) {
                ((HttpSecurity)((DefaultLoginPageConfigurer)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)this.http.csrf().and()).addFilter(new WebAsyncManagerIntegrationFilter()).exceptionHandling().and()).headers().and()).sessionManagement().and()).securityContext().and()).requestCache().and()).anonymous().and()).servletApi().and()).apply(new DefaultLoginPageConfigurer())).and()).logout();
                ClassLoader classLoader = this.context.getClassLoader();
                // 通过 AbstractHttpConfigurer 子类加载过滤器
                List<AbstractHttpConfigurer> defaultHttpConfigurers = SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);
                Iterator var6 = defaultHttpConfigurers.iterator();

                while(var6.hasNext()) {
                    AbstractHttpConfigurer configurer = (AbstractHttpConfigurer)var6.next();
                    this.http.apply(configurer);
                }
            }

            // 执行 configure(HttpSecurity http) 加载用户配置
            this.configure(this.http);
            return this.http;
        }
    }
    
    protected AuthenticationManager authenticationManager() throws Exception {
        if (!this.authenticationManagerInitialized) {
            // 执行 configure(AuthenticationManagerBuilder auth) 加载用户配置
            this.configure(this.localConfigureAuthenticationBldr);
            if (this.disableLocalConfigureAuthenticationBldr) {
                // 构建 AuthenticationManagerBuilder 实例，AuthenticationManagerBuilder 实例拥有 authenticationProviders 属性管理 AuthenticationProvider 实例
                this.authenticationManager = this.authenticationConfiguration.getAuthenticationManager();
            } else {
                this.authenticationManager = (AuthenticationManager)this.localConfigureAuthenticationBldr.build();
            }

            this.authenticationManagerInitialized = true;
        }

        return this.authenticationManager;
    }
}
```

实际上，HttpSecurity 也是基于 filter 实现的，通过 configure 方法配置规则就是配置过滤器。基于 filter，Spring Security 提供以下三个接口用于鉴权：

* Authentication：扩展自 Principal，包含 getIdentity 方法返回一个代表 Principal 标识的 Object（通常是用户名）；getCredentials 方法返回用于验证用户身份的凭证（只在认证过程中使用，认证结束后被擦除）；isAuthenticated 方法返回是否通过认证；setAuthenticated 修改认证结果（通常也只在认证过程中使用）。
* GrantedAuthentication：用以判断用户包含的角色、权限信息。Authentication 接口中的 getAuthorities 方法可用于获取用户的 GrantedAuthentication。
* AuthenticationProvider：认证服务的提供者，包含 authenticate 方法以未认证的 Authentication 作为参数，该方法将把 Authentication 标识为已认证，并返回已认证的但完全不同的 Authentication。

Spring Security 既能使用内建的系统如 CAS、JAAS、LDAP、OpenID 鉴权，又能对接 JDBC 或自己的服务或仓库用于鉴权。

### 结语

因为对 spring 机制的不熟悉以及 Spring Security 应用场景有限，笔者的行文仅点到为止。

### 参考

[Spring高级篇—Spring Security入门原理及实战](http://www.zijin.net/news/tech/1190163.html)
[Spring Boot Security 详解](http://blog.itwolfed.com/blog/14)