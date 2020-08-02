---
title: spring boot 入门：redis 缓存服务
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: e219e00a
date: 2019-03-17 05:50:00
updated: 2019-03-17 05:50:00
---

[redis](http://www.redis.cn/) 可以作为消息中间件、缓存服务器，在集群、分布式环境中有广泛使用，以便于解决共享数据的问题，使单机保持无状态（比如登录状态）。这里只介绍怎样在 Spring Boot 项目中使用 redis 服务。首先需要添加 spring-boot-starter-data-redis、jedis 依赖。[jedis](https://github.com/xetorthio/jedis) 是 redis 官方推荐的 java 连接开发工具。与数据库访问操作中先有 jdbc、后有 mybatis 一样，jedis 也需要配置连接池。本文只简单地介绍如何用 Spring Boot 使用 redis 服务。

首先需要添加 Config 配置文件，以创建 jedis 连接池。[jedis 官方文档](https://github.com/xetorthio/jedis/wiki/Getting-started#using-jedis-in-a-multithreaded-environment)说明了创建 jedis 连接池的必要性，其一可以避免创建大量的 jedis 实例，其二通过 jedis 连接池获取的 jedis 实例是线程安全的。直接获取 jedis 实例并不是线程安全的。通过以下代码也能看出，我们还需要配置 application.yml 文件：

```java
@Configuration
public class RedisConfig {
    // 读取 application.yml 配置
    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private Integer port;

    @Value("${spring.redis.password}")
    private String password;

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.pool")
    public JedisPoolConfig getJedisPoolConfig() {
        return new JedisPoolConfig();
    }

    @Bean
    public ShardedJedisPool shardedJedisPool(){
        // 连接池配置
        JedisPoolConfig jedisPoolConfig = getJedisPoolConfig();
        ArrayList<JedisShardInfo> arrayList = new ArrayList<>();
        JedisShardInfo redisShardInfo = new JedisShardInfo(host, port);
        redisShardInfo.setPassword(password);
        arrayList.add(redisShardInfo);
        // ShardedJedisPool 一般用于 redis 多实例部署，读写分离、主从模式等
        return new ShardedJedisPool(jedisPoolConfig, arrayList);
    }
}
```

基于 lamda 表达式制作 RedisUtils 工具类。有了 RedisUtils 工具类，我们就可以在 controller、service 层访问 redis 缓存数据了。

```java
public interface RedisUtils {
    String set(String key, String value);
    String get(String key);
    Long expire(String key, Integer expire);
    Long del(String key);
}

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