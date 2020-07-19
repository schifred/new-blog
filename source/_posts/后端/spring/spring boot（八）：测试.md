---
title: spring boot（八）：测试
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: b600fbee
date: 2019-11-11 06:00:00
updated: 2019-11-11 06:00:00
---

在 spring mvc 中，可使用 JUnit 作单元测试、Spring Test 作集成测试。测试过程不需要启动项目，可借助 Servlet 相关的模拟对象如 MockMVC、MockHttpServletResquest、MockHttpServletResponse、MockHttpSession 进行模拟。在 spring boot 中，编写测试用例前先添加 spring-boot-starter-test 依赖。

JUnit4 为了保证每个测试方法都是单元测试，是独立的互不影响。所以每个测试方法执行前都会重新实例化测试类，它提供了如下注解：

* @BeforeClass 和 @AfterClass 在类被实例化前后执行，且只执行一次，通常用来初始化或关闭资源
* @Before 和 @After 在每个 @Test 执行前后都会被执行一次
* 使用 @Test 标记测试方法单元，被 @Ignore 标记的测试方法不会被执行

测试脚本编写完成后，可直接运行 run 跑测试用例。

```java
@RunWith(SpringRunner.class)// 运用 junit4 进行测试，SpringRunner.class 等同 SpringJUnit4ClassRunner.class
@SpringBootTest(classes = {YourApplication.class})
public class DemoControllerTest {
    @Autowired
    private MockMvc mockMvc;
  
    @Autowired
    private DemoController demoController;
    
    @Autowired 
    WebApplicationContext wac;// 可注入 WebApplicationContext
    
    @Autowired 
    MockHttpSession session;// 可注入 http session
    
    @Autowired 
    MockHttpServletRequest request;// 可注入 request
    
    @Before
    public void setup() {
      // 初始化 mockMvc
      mockMvc = MockMvcBuilders.webAppContextSetup(this.wac).build();
    }
  
    @Test
    public void testPageController() throws Exception {
      mockMvc.perform(get("/normal"))
          .andExpect(status().isOk())
          .andExpect(view().name("page"))// 预期 view 的名称为 page
          .andExpect(forwardedUrl("/WEB-INF/classes/views/page.jsp"))// 预期页面转向的真正路径为 /WEB-INF/classes/views/page.jsp
          .andExpect(model().attribute("msg", demoService.saySomething()));// 预期 model 里的值为 demoService.saySomething 方法的返回结果
    }

    @Test
    public void testRestController() throws Exception {
      mockMvc.perform(get("/testRest"))
          .andExpect(status().isOk())
          .andExpect(content().contentType("text/plain;charset=UTF-8"))// 期望返回值的媒体类型是 text/plain、UTF-8
          .andExpect(content().string(demoService.saySomething()));// 期望返回值内容为 demoService.saySomething 方法的返回结果
    }
}
```

### 参考

[Spring Boot测试](https://www.jianshu.com/p/76b893f43bdd)