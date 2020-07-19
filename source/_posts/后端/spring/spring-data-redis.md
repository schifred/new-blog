---
title: spring-data-redis
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - redis
keywords: 'redis,spring'
abbrlink: 4502320b
date: 2019-12-01 00:00:00
updated: 2019-12-01 00:00:00
---

redis 缓存数据库通过 redis-cli 工具操控；并支持使用 EVAL、EVALSHA 命令执行脚本。缓存的数据格式支持字符串、哈希、列表、无序集合、有序集合以及 HyperLogLog，支持使用 EXPIRE 设置某键的过期时间、备份数据和恢复数据。redis 本身是基于客户端-服务端模型以及请求-响应协议实现的 TCP 服务，通过管道（pipeline）的技术可以批量发送并处理请求，节省响应时间。redis 支持基于频道的发布、订阅功能，发布者将消息推送到频道上，就会触发该频道的订阅者执行特定的程式。redis 支持事务和基于 WATCH 实现的乐观锁（当 WATCH 监听的键被其他命令修改时，事务将被打断）。通过主键取模或哈希取模，可以在 redis 服务器上使用多个 redis 实例分区存取数据。等等。

### spring-data-redis

spring-data-redis 提供了四种创建 redis 数据库连接的连接工厂，即 JedisConnectionFactory、JredisConnectionFactory、LettuceConnection、SrpConnectionFactory，所有的连接工厂都包含 setHostName、setPort、setPassword、getConnection 方法。各连接工厂的 getConnection 方法用于获取 RedisConnection 接口的各种实现类。通过对接 RedisKeyCommands 等 redis 命令的抽象接口，RedisConnection 接口的实现类封装了操纵 redis 数据库的方法，比如 conn.get(“greeting”.getBytes())，进出 redis 数据库的 key、value 键值对都是字节数组形式。详情可以参看 DefaultedRedisConnection 的实现。

为了规避字节数组，spring-data-redis 以模板形式提供了较高等级的数据访问方案，包含的模板有 RedisTemplate、StringRedisTemplate，这些模板用于对出入库的数据执行序列化操作、获取数据出入库的操纵方法 ValueOperations 实现类等。首先基于 ConnectionFactor，spring-data-redis 提供了 RedisConnectionUtils 用于指定连接工厂、创建连接、关闭连接、创建代理连接、管理连接事务等。其次在使用 RedisConnectionUtils 获取 redis 连接的基础上，RedisTemplate 可以设置不同数据类型的序列化工具；抽象了执行脚本或 lambda 表达式的 execute、executePipeline 方法；抽象了 set、get、delete、hasKey、expire、multi、match 操纵出入库数据的方法；抽象了操纵不同数据类型的 ValueOperations、ListOperations、SetOperations、ZSetOperations、HashOperations、BoundValueOperations、BoundValueOperations、BoundListOperations、BoundSetOperations、BoundZSetOperations、BoundHashOperations 等操作对象。操作对象既可能会调用 RedisTemplate 实例中的 set、get 方法，又可能调用 redisTemplate.execute 执行 lambda 表达式。

```java
public class RedisTemplate<K, V> extends RedisAccessor implements RedisOperations<K, V>, BeanClassLoaderAware {
	@Override
	public RedisSerializer<?> getHashValueSerializer() {
		return hashValueSerializer;
	}

	public void setHashValueSerializer(RedisSerializer<?> hashValueSerializer) {
		this.hashValueSerializer = hashValueSerializer;
	}
  
  // 执行脚本
	@Override
	public <T> T execute(RedisScript<T> script, List<K> keys, Object... args) {
		return scriptExecutor.execute(script, keys, args);
	}

	@Override
	public BoundValueOperations<K, V> boundValueOps(K key) {
		return new DefaultBoundValueOperations<>(key, this);
	}

	@Override
	public ValueOperations<K, V> opsForValue() {

		if (valueOps == null) {
			valueOps = new DefaultValueOperations<>(this);
		}
		return valueOps;
	}
}

class DefaultBoundValueOperations<K, V> extends DefaultBoundKeyOperations<K> implements BoundValueOperations<K, V> {
	private final ValueOperations<K, V> ops;

	DefaultBoundValueOperations(K key, RedisOperations<K, V> operations) {

		super(key, operations);
		this.ops = operations.opsForValue();
	}

	@Override
	public V get() {
		return ops.get(getKey());
	}

	@Override
	public String get(long start, long end) {
		return ops.get(getKey(), start, end);
	}

	@Override
	public void set(V value, long timeout, TimeUnit unit) {
		ops.set(getKey(), value, timeout, unit);
	}

	@Override
	public void set(V value) {
		ops.set(getKey(), value);
	}

	@Override
	public void set(V value, long offset) {
		ops.set(getKey(), value, offset);
	}
}
```

有了 RedisTemplate，我们就可以使用如下的方式在 redis 数据库中存取数据：

```java
// 简单的值
redisTemplate.opsForValue().set(product.getSku(), product);// 使用序列化工具将 product 实体转化成字符串
redisTemplate.opsForValue().get("123456");// 取出 product

// list
redisTemplate.opsForList().rightPush("cart", product);// 尾部插入
redisTemplate.opsForList().leftPush("cart", product);// 顶部插入
Product first = redisTemplate.opsForList().leftPop("cart");// 顶部取值
Product last = redisTemplate.opsForList().rightPop("cart");// 尾部取值
List<Product> products = redisTemplate.opsForList().range("cart", 2, 12);// 范围取值

// set
redisTemplate.opsForSet().add("cart1", product);// 添加
redisTemplate.opsForSet().remove(product);// 移除
redisTemplate.opsForSet().randomMember("cart1");// 随机取元素
List<Product> diff = redisTemplate.opsForSet().difference("cart1", "cart2");// 差集
List<Product> union = redisTemplate.opsForSet().union("cart1", "cart2");// 并集
List<Product> isect = redisTemplate.opsForSet().isect("cart1", "cart2");// 交集

// 嫁接
BoundListOperations<String, Product> cart = redisTemplate.boundListOpts("cart");
cart.rightPush(cart1);
cart.rightPush(cart2);
```

### 参考

[redis 文档中心](http://www.redis.cn/documentation.html)
[RedisTemplate使用方法归纳](https://www.jianshu.com/p/0fa4c100e9a9)