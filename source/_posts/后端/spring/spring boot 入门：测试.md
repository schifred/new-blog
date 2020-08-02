---
title: spring boot 入门：测试
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: b600fbee
date: 2019-03-17 06:00:00
updated: 2019-03-17 06:00:00
---

Spring Boot 项目可以借助 JUnit 作单元测试、Spring Test 作集成测试。编写测试用例前需要添加 spring-boot-starter-test 依赖，这样会加载 JUnit4 依赖。以下是编写测试类时的常用注解（IDEA 可在选中待测试类后，使用 ⇧⌘T(MAC) 或者 Ctrl+Shift+T(Window) 快捷键创建测试类，或者点击 Navigate - Test 面板创建测试类）：

* @SpringBootTest：启动嵌入式服务器并完全初始化应用程序上下文，随后在测试类中可以使用 @Autowired 注解加载 bean。
* @RunWith(SpringRunner.class)：JUnit 提供，意在更改 JUnit 的测试运行器。本例中让 JUnit 运行 Spring 的测试环境，获得 Spring 环境的上下文的支持。SpringRunner 扩展自 SpringJUnit4ClassRunner。

### JUnit

JUnit 为保证每个测试方法都是单元测试，相互独立，互不影响，它会在每个测试方法执行前都会重新实例化测试类。

* @BeforeClass、@AfterClass：在类被实例化前后执行，且只执行一次，通常用来初始化或关闭资源
* @Before、@After：在每个 @Test 执行前后都会被执行一次
* @Test：标记测试方法单元，被 @Ignore 标记的测试方法不会被执行

此外，JUnit 提供了许多断言方法。这些方法失败时会报错，成功时不会打印日志。

* assertTrue(msg, actual)：校验是否为 true
* assertFalse(msg, actual)：校验是否为 false
* assertNotNull(msg, actual)：校验非 null
* assertNull(msg, actual)：校验 null
* assertSame(msg, expected, actual)：是否相同
* assertNotSame(msg, expected, actual)：是否不相同
* assertEquals(msg, expected, actual)：是否相等
* assertArrayEquals(msg, expected, actual)：数组是否相等

附加框架 Hamcrest 提供了 allOf、anyOf、both、containsString、equalTo、everyItem、hasItems、not、sameInstance、startsWith 等 matcher。JUnit 4.4 结合 Hamcrest 提供了一个全新的断言语法 —— assertThat。编码时可以使用 assertThat(new Object(), not(sameInstance(new Object()))) 形式。

测试过程不需要启动项目，可借助 Servlet 相关的模拟对象如 MockMVC、MockHttpServletResquest、MockHttpServletResponse、MockHttpSession 进行模拟。

### controller 单元测试

MockMvc 实现了对 Http 请求的模拟，能够直接使用网络请求的形式触发 Controller 的调用；并提供了一套验证工具。

可以在测试类上添加 @WebMvcTest 注解，这样只有部分 bean 会被扫描到，分别是 @Controller、@ControllerAdvice、@JsonComponent、Filter、WebMvcConfigurer、HandlerMethodArgumentResolver。其他常规的 @Component（包括 @Service、@Repository 等）不会被加载到 Spring 测试环境上下文中。

```java
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultHandlers;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = {DemoApplication.class})
@WebAppConfiguration
public class DemoControllerTest {
    @Autowired 
    WebApplicationContext wac;// 注入 WebApplicationContext
    @Autowired 
    MockHttpSession session;// 注入 http session
    @Autowired 
    MockHttpServletRequest request;// 注入 request

    @Autowired
    private MockMvc mvc;
    
    @Before
    public void setup() {
      mvc = MockMvcBuilders.webAppContextSetup(this.wac).build();// 初始化 mockMvc
      User user = new User("root","123456");
      session.setAttribute("user",user);// 模拟用户
    }
  
    // 页面测试
    @Test
    public void testPage() throws Exception {
      mvc.perform(MockMvcRequestBuilders.get("/index"))
        .andExpect(MockMvcResultMatchers.status().isOk())
        .andExpect(MockMvcResultMatchers.view().name("index"))// 预期 view 的名称为 index
        .andExpect(MockMvcResultMatchers.forwardedUrl("/index"))// 预期页面重定向到 index
        .andExpect(MockMvcResultMatchers.model().attribute("msg", "test"));
    }

    @Test
    public void testJson() throws Exception {
      mockMvc.perform(MockMvcRequestBuilders.get("/json"))
        .andExpect(MockMvcResultMatchers.status().isOk())
        .andExpect(MockMvcResultMatchers.content().contentType("application/json"))// 期望返回 json
        .andExpect(MockMvcResultMatchers.content().string("{test: 123}"))// 期望返回值
        .session(session)
        .andDo(MockMvcResultHandlers.print());// 打印数据
    }

    @Test
    @Transactional// 添加注解以便回滚，数据将不会入库
    public void testPost() throws Exception {
      mockMvc.perform(MockMvcRequestBuilders.post("/update")
          .accept(MediaType.APPLICATION_JSON_UTF8)// org.springframework.http.MediaType
          .content("{test: 123}")
          .session(session)
        ).andExpect(MockMvcResultMatchers.status().isOk())
        .andExpect(MockMvcResultMatchers.status().isOk())
    }
}
```

### service 单元测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {DemoApplication.class})
public class DemoServiceUnitTest {
    @Autowired
    private DemoService demoService;
   
    @Test
    public void getLearn(){
      Demo demo = demoService.selectByKey(1001L);
      Assert.assertThat(demo.getName(), is("测试"));
    }
}
```

### Mockito

当服务的调用链较为复杂时，使用 Mockito 可用于创建模拟对象并提供模拟服务。spring-boot-starter-test 内置 mockito-core。

```java
@RunWith(MockitoJUnitRunner.class)
public class UserServiceTest {
  @Mock
  private UserMapper userMapper;// 模拟 mapper 服务

  @InjectMocks
  private UserServiceImpl userService;// service 调用模拟的 mapper 服务

  @Test
  public void testUpdateUsernameFailed() {
    User user = new User("root", "123456");
    when(userMapper.getById(1001L)).thenReturn(user);// 当调用 mapper 服务时约定返回内容

    assertEquals('same', userService.getById(1001L), user);
  }
}
```

### 常见问题

Q：cannot resolve method get/status
A：手动导入依赖，并编码
```java
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultHandlers;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
// MockMvcRequestBuilders.get
// MockMvcResultMatchers.status
// MockMvcResultMatchers.content
// MockMvcResultHandlers.print
```

Q：Failed to load ApplicationContext 报错
A：测试类文件中添加 @ContextConfiguration(locations= {"classpath*:application.yml","classpath*:logback_config.xml"}) 注解

Q：org.xml.sax.SAXParseException; lineNumber: 1; columnNumber: 1; 前言中不允许有内容
A：application.yml 文件 logback-config.xml 配置不正确；或需移除 @ContextConfiguration(locations= {"classpath*:application.yml", "classpath*:logback-config.xml"}) 注解

Q：测试类自动装配 Controller 失败
A：测试类须添加 @SpringBootTest(classes = {DemoApplication.class}) 注解，声明启动类，详见 SpringBoot中Junit测试注入Bean失败的解决方法