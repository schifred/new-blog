---
title: spring boot 进阶：使用 spring cache
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
keywords: spring
abbrlink: 96b22f49
date: 2019-03-17 05:49:00
updated: 2019-03-17 05:49:00
---

spring cache 基于 aop 机制实现，其本身没有缓存能力，需要指定具体的缓存管理器。如下：

```java
@EnableCaching
@Configuration
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(Arrays.asList(new ConcurrentMapCache("default")));
        return cacheManager;
    }
}
```

spring cache 可用的注解有：

* @Cacheable(value, key)：标注在类或方法上，表示返回结果首先从缓存中获取，一般作用于查询方法。value 定义缓存的名称，key 缓存主键，对应方法的请求参数
* @CachePut(value, key)：更新缓存，一般作用于创建或更新方法
* @CacheEvict(value, key)：清除缓存，一般作用于删除方法
* @Caching：组合多个 @Cacheable、@CachePut、@CacheEvict

```java
@RestController
@RequestMapping("/user")
public class UserController extends BaseController {
    @Cacheable(value = "getUser", key = "#id")
    @GetMapping("/getUser")
    public Result getUser(@RequestParam(name = "id", required = true) String id){
        UserDTO userDTO = userService.getUser(Long.valueOf(id));
        return ResultUtil.success(userDTO);
    }

    @CachePut(value = "userById", key = "#userDTO.id")
    @PostMapping("/createUser")
    public Result createUser(@Valid UserDTO userDTO, BindingResult result){
        if (result.hasErrors()) {
            ResultUtil.error(400, result.getAllErrors().get(0).getDefaultMessage());
        }
        int i = userService.createUser(userDTO);
        return ResultUtil.success(i);
    
    @CacheEvict(value = "userById", key = "#id")
    @PostMapping("/deleteUser")
    public Result deleteUser(@RequestParam(name = "id", required = true) String id){
        int i = userService.delete(Long.valueOf(id));
        return ResultUtil.success(i);
    }
}
```

### key 键生成器

如果 @Cacheable、@CachePut、@CacheEvict 注解上没有指定 key 键，spring cache 会使用 KeyGenerator 自动生成 key 键，我们也可以设置 KeyGenerator 生成策略。具体方法就是定义一个 myKeyGenerator，然后用 @Cacheable(value="userById", keyGenerator="myKeyGenerator") 标注方法。

```java
@EnableCaching
@Configuration
public class CacheConfig {
    @Bean(name = "myKeyGenerator")
    public KeyGenerator keyGenerator(){
        return new KeyGenerator() {
            @Override
            public Object generate(Object target, Method method, Object... params) {
                return method.getName()+"["+Arrays.asList(params).toString() +"]";
            }
        };
    }
}
```

### 对接 redis 缓存

```java
@EnableCaching
@Configuration
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory){
        //缓存配置对象
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();

        redisCacheConfiguration = redisCacheConfiguration.entryTtl(Duration.ofMinutes(30L))// 设置缓存的默认超时时间：30分钟
            .disableCachingNullValues()// 如果是空值，不缓存
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(keySerializer()))// 设置key序列化器
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer((valueSerializer())));  //设置value序列化器

        return RedisCacheManager
                .builder(RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory))
                .cacheDefaults(redisCacheConfiguration).build();
    }

    @Bean(name = "myKeyGenerator")
    public KeyGenerator keyGenerator(){
        return new KeyGenerator() {
            @Override
            public Object generate(Object target, Method method, Object... params) {
                return method.getName()+"["+Arrays.asList(params).toString() +"]";
            }
        };
    }

    @Bean
    public RedisTemplate<String,Object> redisTemplate(RedisConnectionFactory redisConnectionFactory){
        RedisTemplate<String,Object> redisTemplate = new RedisTemplate<>();

        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.setKeySerializer(keySerializer());
        redisTemplate.setHashKeySerializer(keySerializer());
        redisTemplate.setValueSerializer(valueSerializer());
        redisTemplate.setHashValueSerializer(valueSerializer());
        return redisTemplate;
    }

    private RedisSerializer<String> keySerializer() {
        return new StringRedisSerializer();
    }

    private RedisSerializer<Object> valueSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }
}
```