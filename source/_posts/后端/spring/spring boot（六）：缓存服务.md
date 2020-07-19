---
title: spring boot（六）：redis 缓存服务
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 63b17f08
date: 2019-03-17 05:00:00
updated: 2019-03-17 05:00:00
---

1. pom.xml 添加 spring-boot-starter-data-redis、jedis。
2. 添加 config/RedisConfig 文件。

```java
@Configuration
public class RedisConfig {
    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private Integer port;

    @Value("${spring.redis.password}")
    private String password;

    @Bean
    public ShardedJedisPool shardedJedisPool(){
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMinIdle(100);
        jedisPoolConfig.setMaxIdle(1000);
        jedisPoolConfig.setMaxTotal(10000);
        ArrayList<JedisShardInfo> arrayList = new ArrayList<>();
        JedisShardInfo redisShardInfo = new JedisShardInfo(host, port);
        redisShardInfo.setPassword(password);
        arrayList.add(redisShardInfo);
        return new ShardedJedisPool(jedisPoolConfig, arrayList);
    }
}
```

3. 制作 RedisUtils 模块。

```java
// redis/RedisUtils
public interface RedisUtils {
    /**
     * set存数据
     * @param key
     * @param value
     * @return
     */
    String set(String key, String value);

    /**
     * get获取数据
     * @param key
     * @return
     */
    String get(String key);

    /**
     * 设置有效天数
     * @param key
     * @param expire
     * @return
     */
    Long expire(String key, Integer expire);

    /**
     * 移除数据
     * @param key
     * @return
     */
    Long del(String key);
}

// redis/RedisUtilsImpl
@Component
public class RedisUtilsImpl implements RedisUtils {
    @Autowired(required = false)
    private ShardedJedisPool shardedJedisPool;

    private <T> T excute(Function<ShardedJedis, T> func){
        ShardedJedis shardedJedis = shardedJedisPool.getResource();
        return func.apply(shardedJedis);
    }

    @Override
    public String set(final String key, final String value) {
        return excute(e -> e.set(key, value));
    }

    @Override
    public String get(final String key) {
        return excute(e -> e.get(key));
    }

    @Override
    public Long expire(final String key, Integer expire) {
        return excute(e -> e.expire(key, expire));
    }

    @Override
    public Long del(final String key) {
        return excute(e -> e.del(key));
    }
}
```

4. 使用

```java
@RestController
@RequestMapping("/redis")
public class RedisController {
    @Resource
    RedisUtils redisUtils;

    @RequestMapping(value = "/set/{str}", method = RequestMethod.GET)
    public String set(@PathVariable("str") String str){
        return redisUtils.set("test", str);
    }

    @RequestMapping(value = "/get", method = RequestMethod.GET)
    public String get(){
        return redisUtils.get("test");
    }
}
```

### 参考

[MAC使用homeBrew安装Redis](https://www.jianshu.com/p/e1e5717049e8)