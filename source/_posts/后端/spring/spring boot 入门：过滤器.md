---
title: spring boot 入门：过滤器
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 68ab87cc
date: 2019-03-17 02:10:00
updated: 2019-03-17 02:10:00
---

常规的控制器只会对特定的请求地址作处理。当我们需要对一类请求进行处理时，过滤器就隆重登场了。很显然，过滤器在用户信息校验、验权等场景中会显得分外有效。过滤器是 Servlet 机制提供的。在 Spring Boot 中，一个类只要实现 Filter 接口，就会对请求进行拦截。这就提出了以下几个问题：过滤器会对哪些请求进行拦截；多个过滤器中，何者优先处理等。FilterConfig 类即作用于以上问题：

```java
@Configuration
public class FilterConfig {
    @Autowired
    private SessionFilter sessionFilter;

    @Bean
    public FilterRegistrationBean registerAuthFilter() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(sessionFilter);
        registration.addUrlPatterns("/*");
        registration.setName("sessionFilter");
        registration.setOrder(1);
        return registration;
    }
}

@Component
public class SessionFilter implements Filter {
    private static final Logger log = LoggerFactory.getLogger(SessionFilter.class);

    private static final String session_user_key = "user_key";
    private static final String api_should_login = "/api";

    String[] excludeUrls = new String[]{"/user/login","/user/register","/user/getUserInfo"};

    @Override
    public void init(FilterConfig filterConfig) throws ServletException { }

    @Override
    public void destroy() { }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        HttpSession session = request.getSession(false);// 获取当前请求的 session
        String uri = request.getRequestURI();// 获取访问路径
        String servletPath = request.getServletPath();// 获取访问路径，不带域名协议

        boolean needLogin = shouldLogin(uri);
        if (!needLogin) {
            chain.doFilter(request, response);
        } else {
            if(null != session && null != session.getAttribute(session_user_key)){
                chain.doFilter(request, response);
            }else{
                if (servletPath.indexOf(api_should_login) == 0){
                    Result result = ResultUtil.error(403, "should login");
                    response.getWriter().write(JSON.toJSONString(result));
                } else {
                    response.sendRedirect("/login");
                }
            }
        }
    }

    public boolean shouldLogin(String uri) {
        for (String excludeUrl : excludeUrls) {
            if(uri.matches(excludeUrl)) {
                return false;
            }
        }

        return true;
    }
}
```

除了通过 FilterRegistrationBean 注册过滤器外，过滤器也可以通过直接加注解的方式添加。如下：

```java
@WebFilter(filterName = "TestFilter", urlPatterns = { "/*" })
public class TestFilter implements Filter {
	@Override
	public void init(FilterConfig filterConfig) throws ServletException { }

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
		response.setCharacterEncoding("UTF-8");
		chain.doFilter(request, response);
	}

	@Override
	public void destroy() { }
}
```

更多内容可戳 [@WebFilter注解](https://www.cnblogs.com/ooo0/p/10360952.html)。