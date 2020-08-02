---
title: spring boot 入门：使用 RestTemplate 发送 http 请求
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 7d49460
date: 2019-03-17 05:53:00
updated: 2019-03-17 05:53:00
---

如果要向外部系统发送 http、https 请求，可以借助 apache 的 http-client 创建 HttpGet、HttpPost 实例实现，或者借助 rest-template。使用 http-client 编码较为繁琐（需要 close），rest-template 较为简单，且不需要引入额外的依赖，其简单使用示例如下：

```java
@RestController
@RequestMapping("/rest-template")
public class RestTemplateController extends BaseController {
    @GetMapping("/getUser")
    public Result getUser(@RequestParam(name = "id", required = true) String id){
        RestTemplate restTemplate = new RestTemplate();
        UserDTO userDTO = restTemplate.getForObject("http://localhost:8080/user/getUser?id=" + id, UserDTO.class);
        return ResultUtil.success(userDTO);
    }
}
```

### 调用方法

rest-template 支持的方法有：

* getForObject：get 请求，按需求转成 POJO
* getForEntity：get 请求，返回 ResponseEntity
* postForObject：post 请求，按需求转成 POJO
* postForEntity：post 请求，返回 ResponseEntity
* exchange：可以指定请求方式、消息头，返回 ResponseEntity
* excute：同 exchange，可以使用 lamda 表达式指定响应实体

### 连接方式、消息转换器

我们也可以通过 RestTemplateConfig 配置类创建 bean 实例，然后在 controller 中加载。这样全局就只有一个 RestTemplate 实例，便于设置消息转换器（用于转换响应内容格式，如 json 转换器、设置字符为 utf-8 编码）、底层连接方式。rest-template 默认使用 java.net.HTTP 连接，也支持 Netty 或 apache 的 http-client 连接方式。

```java
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate getRestTemplate(){
        RestTemplate restTemplate = new RestTemplate(getSimpleClientHttpRequestFactory());
        return restTemplate;
    }

    private void applyConverter(RestTemplate restTemplate){
        // 替换转换器
        List<HttpMessageConverter<?>> messageConverters = restTemplate.getMessageConverters();
        messageConverters.remove(6);
        messageConverters.add(6, new GsonHttpMessageConverter());
    }

//    private ClientHttpRequestFactory getClientHttpRequestFactory(){
//         // 使用 apache 的 http-client
//         RequestConfig config = RequestConfig.custom().setConnectionRequestTimeout(10000).setConnectTimeout(10000).setSocketTimeout(30000).build();
//         HttpClientBuilder builder = HttpClientBuilder.create().setDefaultRequestConfig(config).setRetryHandler(new DefaultHttpRequestRetryHandler(5, false));
//         HttpClient httpClient = builder.build();
//         ClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);
//         return requestFactory;
//    }

    private ClientHttpRequestFactory getSimpleClientHttpRequestFactory(){
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setReadTimeout(5000);
        factory.setConnectTimeout(5000);
        return factory;
    }
}
```

### 拦截器

rest-template 支持设置拦截器，通常不必作全局拦截器处理。如下示例来自 [RestTemplate](https://www.jianshu.com/p/90ec27b3b518)：

```java
public class TokenInterceptor implements ClientHttpRequestInterceptor
{
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException
    {
        String checkTokenUrl = request.getURI().getPath();
        int ttTime = (int) (System.currentTimeMillis() / 1000 + 1800);
        String methodName = request.getMethod().name();
        String requestBody = new String(body);
        String token = TokenHelper.generateToken(checkTokenUrl, ttTime, methodName, requestBody);// 生成令牌
        request.getHeaders().add("X-Auth-Token",token);

        return execution.execute(request, body);
    }
}

RestTemplate restTemplate = new RestTemplate();
restTemplate.getInterceptors().add(new TokenInterceptor());
```

### https 请求

使用 rest-template 发送 https 请求仍是基于修改连接方式，[RestTemplate发送HTTP、HTTPS请求
](https://www.cnblogs.com/softidea/p/10663849.html) 介绍了如下方法。

```java
public class HttpClientUtil {

    public static CloseableHttpClient acceptsUntrustedCertsHttpClient() throws KeyStoreException, NoSuchAlgorithmException, KeyManagementException {
        HttpClientBuilder b = HttpClientBuilder.create();

        // setup a Trust Strategy that allows all certificates.
        //
        SSLContext sslContext = new SSLContextBuilder().loadTrustMaterial(null, new TrustStrategy() {
            @Override
            public boolean isTrusted(X509Certificate[] arg0, String arg1) throws CertificateException {
                return true;
            }
        }).build();
        b.setSSLContext(sslContext);

        // don't check Hostnames, either.
        //      -- use SSLConnectionSocketFactory.getDefaultHostnameVerifier(), if you don't want to weaken
        HostnameVerifier hostnameVerifier = NoopHostnameVerifier.INSTANCE;

        // here's the special part:
        //      -- need to create an SSL Socket Factory, to use our weakened "trust strategy";
        //      -- and create a Registry, to register it.
        //
        SSLConnectionSocketFactory sslSocketFactory = new SSLConnectionSocketFactory(sslContext, hostnameVerifier);
        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("http", PlainConnectionSocketFactory.getSocketFactory())
                .register("https", sslSocketFactory)
                .build();

        // now, we create connection-manager using our Registry.
        //      -- allows multi-threaded use
        PoolingHttpClientConnectionManager connMgr = new PoolingHttpClientConnectionManager( socketFactoryRegistry);
        connMgr.setMaxTotal(200);
        connMgr.setDefaultMaxPerRoute(100);
        b.setConnectionManager( connMgr);

        // finally, build the HttpClient;
        //      -- done!
        CloseableHttpClient client = b.build();

        return client;
    }
}


@Configuration
public class RestTemplateConfig {
    @Bean(name = "httpsRestTemplate")
    public RestTemplate httpsRestTemplate(HttpComponentsClientHttpRequestFactory httpsFactory) {
        RestTemplate restTemplate = new RestTemplate(httpsFactory);
        restTemplate.setErrorHandler(
                new ResponseErrorHandler() {
                    @Override
                    public boolean hasError(ClientHttpResponse clientHttpResponse) {
                        return false;
                    }

                    @Override
                    public void handleError(ClientHttpResponse clientHttpResponse) {
                        // 默认处理非200的返回，会抛异常
                    }
                });
        return restTemplate;
    }

    @Bean(name = "httpsFactory")
    public HttpComponentsClientHttpRequestFactory httpComponentsClientHttpRequestFactory()
            throws Exception {
        CloseableHttpClient httpClient = HttpClientUtil.acceptsUntrustedCertsHttpClient();
        HttpComponentsClientHttpRequestFactory httpsFactory =
                new HttpComponentsClientHttpRequestFactory(httpClient);
        httpsFactory.setReadTimeout(40000);
        httpsFactory.setConnectTimeout(40000);
        return httpsFactory;
    }
}
```