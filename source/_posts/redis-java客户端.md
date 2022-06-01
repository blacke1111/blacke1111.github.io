---
title: redis java客户端
date: 2022-05-16 17:22:42
categories: 
---



lettuce：

Lettuce是基于Netty实现的，支持同步、异步和响应式编程方式，并且是线程安全的。支持Redis的哨兵模式、集群模式和管道模式。



# jedis :

以Redis命令作为方法名称，学习成本低，简单实用。但是Jedis实例是线程不安全的，多线程环境下需要基于连接池来使用

jedis测试案例：

**创建maven工程**

**导入pom**

```xml
<dependencies>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.7.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

编写测试代码:

```java
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import redis.clients.jedis.Jedis;
import java.util.Map;

public class JedisTest {
    private Jedis jedis;

    @BeforeEach
    void setup(){
        jedis=new Jedis("192.168.32.3",6379);
        jedis.auth("123456");
        jedis.select(0);
    }

    @Test
    void test1(){
        String result = jedis.set("name", "虎哥");
        System.out.println("result = "+result);
        String name=jedis.get("name");
        System.out.println("name ="+name);
    };

    @Test
    void testHash(){
        jedis.hset("user:1","name","Jack");
        jedis.hset("user:1","age","21");

        Map<String, String> stringStringMap = jedis.hgetAll("user:1");
        System.out.println(stringStringMap);
    }

    @AfterEach
    void tearDown(){
        if (jedis!=null){
            jedis.close();
        }
    }
}

```

## Jedis线程池：

Jedis本身是线程不安全的，并且频繁的创建和销毁连接会有性能损耗，因此我们推荐大家使用Jedis连接池代替Jedis的直连方式。

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;
public class JedisConnectionFactory {
    private  static  final JedisPool jedisPool;
    static {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        //最大连接数
        jedisPoolConfig.setMaxTotal(8);
        //最大空闲数
        jedisPoolConfig.setMaxIdle(8);
        //最大超时时间
        jedisPoolConfig.setMaxWaitMillis(200);
        jedisPool=new JedisPool(jedisPoolConfig,"192.168.150.101",6379,1000,"123456");
    }

    public static Jedis getJedis(){
        return  jedisPool.getResource();
    }
}
```

# SpringDataRedis:

SpringData是Spring中数据操作的模块，包含对各种数据库的集成，其中对Redis的集成模块就叫做SpringDataRedis,官网地址：https://spring.io/projects/spring-data-redis

* 提供了对不同Redis客户端的整合（Lettuce和Jedis）
* 提供了RedisTemplate统一API来操作Redis
* 支持Redis的发布订阅模型
* 支持Redis哨兵和Redis集群
* 支持基于Lettuce的响应式编程
* 支持基于JDK、JSON、字符串、Spring对象的数据序列化及反序列化
* 支持基于Redis的JDKCollection实现

SpringDataRedis中提供了RedisTemplate工具类，其中封装了各种对Redis的操作。并且将不同数据类型的操作API封装到了不同的类型中：

![image-20220516181020393](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220516181020393.png)

## 快速入门：

pom

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

yaml:

```yaml
spring:
  redis:
    host: 192.168.32.3
    port: 6379
    password: 123456
    lettuce:
      pool:
        max-active: 8 # 最大连接
        max-idle: 8 # 最大空闲连接
        min-idle: 0 # 最小空闲连接
        max-wait: 100 # 连接等待时间

```

测试单元内中：

```java
@SpringBootTest
class SpringdataRedisDemoApplicationTests {

    @Autowired
    private RedisTemplate redisTemplate;
    @Test
    void testString() {
        redisTemplate.opsForValue().set("name","redisTemplate");
        //获取
        System.out.println("name = "+redisTemplate.opsForValue().get("name"));
    }

}
```

## RedisTemplate

RedisTemplate可以接收任意Object作为值写入Redis，只不过写入前会把Object序列化为字节形式，默认是采用JDK序列化，得到的结果是这样的：

![image-20220516183610818](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220516183610818.png)

缺点：

* 可读性差
* 内存占用较大

我们可以自定义RedisTemplate的序列化方式，代码如下：

```java
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
throws UnknownHostException {
    // 创建Template
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
    // 设置连接工厂
    redisTemplate.setConnectionFactory(redisConnectionFactory);
    // 设置序列化工具
    GenericJackson2JsonRedisSerializer jsonRedisSerializer =
    new GenericJackson2JsonRedisSerializer();
    // key和 hashKey采用 string序列化
    redisTemplate.setKeySerializer(RedisSerializer.string());
    redisTemplate.setHashKeySerializer(RedisSerializer.string());
    // value和 hashValue采用 JSON序列化
    redisTemplate.setValueSerializer(jsonRedisSerializer);
    redisTemplate.setHashValueSerializer(jsonRedisSerializer);
    return redisTemplate;
}
```

尽管JSON的序列化方式可以满足我们的需求，但依然存在一些问题，如图：

![image-20220516191558052](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220516191558052.png)

为了在反序列化时知道对象的类型，JSON序列化器会将类的class类型写入json结果中，存入Redis，会带来额外的内存开销。

为了节省内存空间，我们并不会使用JSON序列化器来处理value，而是统一使用String序列化器，要求只能存储String类型的key和value。当需要存储Java对象时，手动完成对象的序列化和反序列化。

![image-20220516191624319](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220516191624319.png)

## StringRedisTemplate

Spring默认提供了一个StringRedisTemplate类，它的key和value的序列化方式默认就是String方式。省去了我们自定
义RedisTemplate的过程：

```java
private StringRedisTemplate stringRedisTemplate;
    // JSON工具
    private static final ObjectMapper mapper = new ObjectMapper();
    @Test
    void testStringTemplate() throws JsonProcessingException {
    // 准备对象
    User user = new User("虎哥", 18);
    // 手动序列化
    String json = mapper.writeValueAsString(user);
    // 写入一条数据到redis
    stringRedisTemplate.opsForValue().set("user:200", json);
    // 读取数据
    String val = stringRedisTemplate.opsForValue().get("user:200");
    // 反序列化
    User user1 = mapper.readValue(val, User.class);
    System.out.println("user1 = " + user1);
}
```

RedisTemplate的两种序列化实践方案：
方案一：

1. 自定义RedisTemplate
2. 修改RedisTemplate的序列化器为GenericJackson2JsonRedisSerializer
方案二：
1. 使用StringRedisTemplate
2. 写入Redis时，手动把对象序列化为JSON
3. 读取Redis时，手动把读取到的JSON反序列化为对象
