---
title: Redis 秒杀业务实战
date: 2022-05-19 22:01:21
categories: Redis
---

#  优惠卷秒杀

## 	全局唯一id

ID的组成部分：

* 符号位：1bit，永远为0
* 时间戳：31bit，以秒为单位，可以使用69年
* 序列号：32bit，秒内的计数器，支持每秒产生2^32个不同ID
* 对应代码

```java
    private static  final  long BEGIN_TIMESTAMP=1640995200L;
    private static final  int COUNT_BITS=32;
    public long nextId(String keyPrefix){
        //1 生成时间戳

        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);

        long timestamp = nowSecond - BEGIN_TIMESTAMP;


        //2 生成序列号
        //2.1 获取到日期，精确到天
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        //2.2 自增长
        //如果对应的key值不存在 会返回1 所以永远不会空指针
        long increment = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
        long left = nowSecond << COUNT_BITS;

        //3 拼接并返回
        return left|increment;
    }
```

## 实现优惠券秒杀下单

![image-20220521142801253](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521142801253.png)

```java
@Resource
    private ISeckillVoucherService seckillVoucherService;

	//自定义订单id生成工具类
    @Resource
    private RedisWorker redisWorker;
    @Override
    @Transactional
    public Result seckillVoucher(Long voucherId) {
        //1 查询优惠卷
        SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
        //2 判断秒杀是否开始
        LocalDateTime beginTime = seckillVoucher.getBeginTime();
        if (beginTime.isAfter(LocalDateTime.now())){
            //秒杀没有开始
            return Result.fail("秒杀还没开始！");
        }
        //3 判断秒杀是否结束
        if (seckillVoucher.getEndTime().isBefore(LocalDateTime.now())){
            //秒杀已经结束
            return Result.fail("秒杀已经结束!");
        }
        //4 判断库存是否充足
        Integer stock = seckillVoucher.getStock();
        if (stock<1){
            return Result.fail("库存不足!!");
        }
        //5 扣减库存
        boolean success = seckillVoucherService.update().setSql("stock=stock-1").eq("voucher_id", voucherId).update();
        if (!success){
            //扣减失败
            return Result.fail("库存不足！");
        }
        //6 创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //订单id
        long orderId = redisWorker.nextId("order");
        voucherOrder.setId(orderId);
        //用户id
        Long userId = UserHolder.getUser().getId();
        voucherOrder.setUserId(userId);
        //代金卷id
        voucherOrder.setVoucherId(voucherId);

        this.save(voucherOrder);
        //7 返回订单id
        return Result.ok(orderId);
    }
```

## 超卖问题:

![image-20220521151308657](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521151308657.png)

**库存超卖**

解决方案：

**悲观锁**

认为线程安全问题一定会发生，因此在操作数据之前先获取锁，确保线程串行执行。

* 例如Synchronized、Lock都属于悲观锁

**乐观锁**

认为线程安全问题不一定会发生，因此不加锁，只是在更新数据时去判断有没有其它线程对数据做了修改。

* 如果没有修改则认为是安全的，自己才更新数据。
* 如果已经被其它线程修改说明发生了安全问题，此时可以重试或异常

乐观锁的关键是判断之前查询得到的数据是否有被修改过，常见的方式有两种：

**版本号法:**

![image-20220521151749737](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521151749737.png)

用数据本身 作比较 实现乐观锁 简称**CAS**

![image-20220521152038846](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521152038846.png)

```java
    @Override
    @Transactional
    public Result seckillVoucher(Long voucherId) {
        //1 查询优惠卷
        SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
        //2 判断秒杀是否开始
        LocalDateTime beginTime = seckillVoucher.getBeginTime();
        if (beginTime.isAfter(LocalDateTime.now())){
            //秒杀没有开始
            return Result.fail("秒杀还没开始！");
        }
        //3 判断秒杀是否结束
        if (seckillVoucher.getEndTime().isBefore(LocalDateTime.now())){
            //秒杀已经结束
            return Result.fail("秒杀已经结束!");
        }
        //4 判断库存是否充足
        Integer stock = seckillVoucher.getStock();
        if (stock<1){
            return Result.fail("库存不足!!");
        }
        //5 扣减库存
        boolean success = seckillVoucherService.update()
                .setSql("stock=stock-1").eq("voucher_id", voucherId).gt("stock",0) //只用判断库存大于0就可以了
                .update();
        if (!success){
            //扣减失败
            return Result.fail("库存不足！");
        }
        //6 创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //订单id
        long orderId = redisWorker.nextId("order");
        voucherOrder.setId(orderId);
        //用户id
        Long userId = UserHolder.getUser().getId();
        voucherOrder.setUserId(userId);
        //代金卷id
        voucherOrder.setVoucherId(voucherId);

        this.save(voucherOrder);
        //7 返回订单id
        return Result.ok(orderId);
    }
```

还有一种 **分段锁的优化方案 把库存分为多个表 用来抢的时候在多个表中同时进行 这样效率大大提高**

## 一人一单:

需求:**修改秒杀业务，要求同一个优惠券，一个用户只能下一单**

![image-20220521153419943](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521153419943.png)

```java
//线程不安全     
   @Override
    @Transactional
    public Result seckillVoucher(Long voucherId) {
        //1 查询优惠卷
        SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
        //2 判断秒杀是否开始
        LocalDateTime beginTime = seckillVoucher.getBeginTime();
        if (beginTime.isAfter(LocalDateTime.now())){
            //秒杀没有开始
            return Result.fail("秒杀还没开始！");
        }
        //3 判断秒杀是否结束
        if (seckillVoucher.getEndTime().isBefore(LocalDateTime.now())){
            //秒杀已经结束
            return Result.fail("秒杀已经结束!");
        }

        //4 判断库存是否充足
        Integer stock = seckillVoucher.getStock();
        if (stock<1){
            return Result.fail("库存不足!!");
        }
        //一人一单 业务
        //用户id
        Long userId = UserHolder.getUser().getId();
        //6.1 查询用户订单
        int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
        //6.2 判断是否存在
        if (count >0 ){
            //用户已经购买
            return Result.fail("用户已经购买过了！！");
        }
        //5 扣减库存
        boolean success = seckillVoucherService.update()
                .setSql("stock=stock-1").eq("voucher_id", voucherId).gt("stock",0) //只用判断库存大于0就可以了
                .update();
        if (!success){
            //扣减失败
            return Result.fail("库存不足！");
        }
        //6 创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //订单id
        long orderId = redisWorker.nextId("order");
        voucherOrder.setId(orderId);
        //用户id
        voucherOrder.setUserId(userId);
        //代金卷id
        voucherOrder.setVoucherId(voucherId);

        this.save(voucherOrder);
        //7 返回订单id
        return Result.ok(orderId);
    }
```

```java
//单体架构线程安全
//1.每个用户线程进来拿到的id是新创建的对象，这样每个线程拿到的所都不是同一个对象。所以必须要使用常量池中的UserId字符串对象作为加锁对象。而且必须锁住的是这个方法 因为这一个方法为一个事务 不能在事务结束之前让其他线程拿到锁对象。 要在事务提交完后释放锁

//2.springboot 做的事务代理 用的是当前对象的代理对象做动态代理 实现事务操作 为了不让事务失效 要做如下处理 
//并且需要添加依赖 
//<dependency>
//  <groupId>org.aspectj</groupId>
//  <artifactId>aspectjweaver</artifactId>
//</dependency>
//还有在启动类上 加上@EnableAspectJAutoProxy(exposeProxy = true)注解 暴露代理对象
synchronized (userId.toString().intern()) {
    //获取代理对象(对象
            IVoucherOrderService proxy = (IVoucherOrderService)AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId,userId);
}
@Transactional
    public  Result createVoucherOrder(Long voucherId,Long userId){

        //5一人一单 业务
        //用户id
            //5.1 查询用户订单
            int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
            //5.2 判断是否存在
            if (count > 0) {
                //用户已经购买
                return Result.fail("用户已经购买过了！！");
            }
            //6 扣减库存
            boolean success = seckillVoucherService.update()
                    .setSql("stock=stock-1").eq("voucher_id", voucherId).gt("stock", 0) //只用判断库存大于0就可以了
                    .update();
            if (!success) {
                //扣减失败
                return Result.fail("库存不足！");
            }

            //7 创建订单
            VoucherOrder voucherOrder = new VoucherOrder();
            //订单id
            long orderId = redisWorker.nextId("order");
            voucherOrder.setId(orderId);
            //用户id
            voucherOrder.setUserId(userId);
            //代金卷id
            voucherOrder.setVoucherId(voucherId);

            this.save(voucherOrder);
            //7 返回订单id
            return Result.ok(orderId);
        }
```

通过加锁可以解决在单机情况下的一人一单安全问题，**但是在集群模式下就不行了。**
1. 我们将服务启动两份，端口分别为8081和8082：

![image-20220521160835557](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521160835557.png)

2. 然后修改nginx的conf目录下的nginx.conf文件，配置反向代理和负载均衡

   ![image-20220521160846790](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521160846790.png)

现在，用户请求会在这两个节点上负载均衡，再次测试下是否存在线程安全问题。

使用postman发送2个请求打到8081和8082这连个服务上在第一个端点停住 ，然后都打到第二个断点这样 再放行 **就发生了重复下单**

因为对应2个springboot服务 使用的是2个不同的jvm 底层就会有不同的堆栈 这样我们**拿不到的锁就不是同一个对象**

![image-20220521162519399](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521162519399.png)

**解决方案：**

## **分布式锁**

### 什么是分布式锁

分布式锁：满足分布式系统或集群模式下多进程可见并且互斥的锁。

### 分布式锁的实现

分布式锁的核心是实现多进程之间互斥，而满足这一点的方式有很多，常见的有三种：

![image-20220521163610118](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521163610118.png)

### 基于redis实现分步式锁

![image-20220521173010163](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521173010163.png)

### 获取锁：

锁接口:

```java
public interface ILock {
    /**
     * 尝试获取锁
     * @param timeoutSec 锁持有的超时时间，过期后自动释放
     * @return true代表获取锁成功; false代表获取锁失败
     */
    boolean tryLock(long timeoutSec);

    void unlock();
}

```



互斥：确保只能有一个线程获取锁

非阻塞： 尝试一次，成功返回true，失败返回false

```
#添加锁,NX是互斥，EX是设置超时时间
SET lock thread NX EX 10
```

```java
@Override
    public boolean tryLock(long timeoutSec) {
        //获取线程标识
        long threadId = Thread.currentThread().getId();
        Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId+"", timeoutSec, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(success);
    }
```

### 释放锁：

```
#释放锁
DEL key
```

```java
 @Override
    public void unlock() {
        stringRedisTemplate.delete(KEY_PREFIX + name);
    }
```

分步式锁实现:

```java
//分步式架构加锁
        //获取锁对象
        SimpleRedisLock lock = new SimpleRedisLock("order:" + userId, stringRedisTemplate);
        boolean isLock = lock.tryLock(1200);
        //判断是否获取锁成功
        if (!isLock){
            //获取锁失败, 返回错误或重试
            return Result.fail("一个人只允许下一单");
        }
        try {
            IVoucherOrderService proxy = (IVoucherOrderService)AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId,userId);
        } finally {
            lock.unlock();
        }
```

### 存在的问题：

![image-20220521175240533](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220521175240533.png)

需求：修改之前的分布式锁实现，满足：
1. 在获取锁时存入线程标示（可以用UUID表示）
2. 在释放锁时先获取锁中的线程标示，判断是否与当前线程标示一致
  * 如果一致则释放锁
  * 如果不一致则不释放锁

```java
public class SimpleRedisLock  implements  ILock{

    private static  final String KEY_PREFIX="lock:";
    private static  final String ID_PREFIX= UUID.randomUUID().toString(true)+"-";
    private String name;

    public SimpleRedisLock(String name, StringRedisTemplate stringRedisTemplate) {
        this.name = name;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    private StringRedisTemplate stringRedisTemplate;

    @Override
    public boolean tryLock(long timeoutSec) {
        //获取线程标识
        String threadId = ID_PREFIX+Thread.currentThread().getId();
        Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(success);
    }

    @Override
    public void unlock() {

        //获取线程标识
        String threadId = ID_PREFIX+Thread.currentThread().getId();
        String value = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
        if (threadId.equals(value)){
            stringRedisTemplate.delete(KEY_PREFIX + name);
        }
    }
}
```

**阻塞的原因可能是jvm的垃圾回收机制 STW  暂停了用户线程**

![image-20220523140003619](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220523140003619.png)

### 原子性问题：

需要把获取锁和释放锁变成一个原子性操作

### Redis的Lua脚本

Redis提供了Lua脚本功能，在一个脚本中编写多条Redis命令，确保多条命令执行时的原子性。Lua是一种编程语言，它的基本语法大家可以参考网站：https://www.runoob.com/lua/lua-tutorial.html
这里重点介绍Redis提供的调用函数，语法如下：

```lua
# 执行redis命令
redis.call('命令名称', 'key', '其它参数', ...)
```

例如，我们要执行set name jack，则脚本是这样：

```lua
# 执行 set name jack
redis.call('set', 'name', 'jack')
```

例如，我们要先执行set name Rose，再执行get name，则脚本如下

```lua
redis.call('set', 'name', 'jack')
# 再执行 get name
local name = redis.call('get', 'name')
# 返回
return name
```

写好脚本以后，需要用Redis命令来调用脚本，调用脚本的常见命令如下：

![image-20220523141414893](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220523141414893.png)

例如，我们要执行 redis.call('set', 'name', 'jack') 这个脚本，语法如下：

```java
//调用脚本 0 脚本需要的key类型的参数个数
EVAL "return redis.call('set','name','jack')" 0
```

如果脚本中的key、value不想写死，可以作为参数传递。key类型参数会放入KEYS数组，其它参数会放入ARGV数组，在脚
本中可以从KEYS和ARGV数组获取这些参数：

![image-20220523142226655](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220523142226655.png)

基于Redis的分布式锁
释放锁的业务流程是这样的：
1. 获取锁中的线程标示

2. 判断是否与指定的标示（当前线程标示）一致

3. 如果一致则释放锁（删除）

4. 如果不一致则什么都不做

  

如果用Lua脚本来表示则是这样的：

```lua
-- 获取锁中的线程标识 get key
local id= redis.call('get',KEYS[1])
-- 比较线程标识与锁中的标识是否一致
if(id==ARGV[1]) then
    --释放锁
    redis.call('del',KEYS[1])
end
return 0
```

需求：基于Lua脚本实现分布式锁的释放锁逻辑
提示：RedisTemplate调用Lua脚本的API如下：

![image-20220523143217464](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220523143217464.png)

![image-20220523143245966](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220523143245966.png)

解锁实现:

```java
private  static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT=new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }
@Override
    public void unlock() {
        //调用lua脚本
        stringRedisTemplate.execute(UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX+name),
                ID_PREFIX+Thread.currentThread().getId());
    }
```

基于Redis的分布式锁实现思路：
• 利用set nx ex获取锁，并设置过期时间，保存线程标示
• 释放锁时先判断线程标示是否与自己一致，一致则删除锁
特性：
• 利用set nx满足互斥性
• 利用set ex保证故障时锁依然能释放，避免死锁，提高安全性
• 利用Redis集群保证高可用和高并发特性

### 基于setnx实现的分布式锁存在下面的问题

1. 不可重入：同一个线程无法多次获取同一把锁
2. 不可重试：获取锁只尝试一次就返回false，没有重试机制
3. 超时释放：锁超时释放虽然可以避免死锁，但如果是业务执行耗时较长，也会导致锁释放，存在安全隐患
4. 主从一致性：如果Redis提供了主从集群，主从同步存在延迟，当主宕机时，如果从并同步主中的锁数据，则会出现锁实现

### Redisson

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务，其中就包含了各种分布式锁的实现。

![image-20220523150103686](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220523150103686.png)

#### Redisson入门：

引入依赖：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.13.6</version>
</dependency>
```

2.配置Redisson客户端

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedissonClient redissonClient() {
// 配置类
        Config config = new Config();
// 添加redis地址，这里添加了单点的地址，也可以使用config.useClusterServers()添加集群地址
        config.useSingleServer().setAddress("redis://192.168.150.101:6379").setPassowrd("123321");
// 创建客户端
        return Redisson.create(config);
    }
}
```

3.使用Rediss的分步式锁

```java
@Resource
        private RedissonClient redissonClient;
        @Test
        void testRedisson() throws InterruptedException {
            // 获取锁（可重入），指定锁的名称
            RLock lock = redissonClient.getLock("anyLock");
            // 尝试获取锁，参数分别是：获取锁的最大等待时间（期间会重试），锁自动释放时间，时间单位
            boolean isLock = lock.tryLock(1, 10, TimeUnit.SECONDS);
            // 判断释放获取成功
            if(isLock){
                try {
                    System.out.println("执行业务");
                }finally {
            // 释放锁
                    lock.unlock();
                }
            }
        }
```

#### 重试问题解决:

```java
// 等待获取锁中  解决了重试问题
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException 
```

第一次获取锁失败，当我们的等待时间没有超时时 当前线程就会**订阅** 那个锁 ，等待锁释放时发布消息给他订阅的客户端。

![image-20220524163011905](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524163011905.png)

如果等待锁的时间 依旧没有超过最长等待时间 那么就会进入while循环尝试第一次 再次获取锁

![image-20220524163508882](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524163508882.png)

while通过信号量来知道，是否锁已经被释放。

![image-20220524163742049](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524163742049.png)

#### 锁释放超时续约：

在尝试获取锁成功后会 调用scheduleExpirationRenewal（threadId）来开启 一个线程刷新锁的有效期。

![image-20220524164701439](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524164701439.png)

ExpirationEntry存放 线程的id 和定时任务。

![image-20220524164919728](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524164919728.png)

**开启一个线程 更新锁的有效期 时间是 默认超时时间的三分之一**，只有当ExpirationEntry中没有线程id，也就是释放锁的时候不在更新有效期 结束递归调用 return;

![image-20220524164522912](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524164522912.png)

解锁成功流程后 回调函数中 取消更新 ，**也就是在ExpirationEntry 删除自己线程的Id**

![image-20220524165311770](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524165311770.png)

![image-20220524165344696](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524165344696.png)

![image-20220524172217539](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524172217539.png)



Redisson分布式锁原理：

* 可重入：利用hash结构记录线程id和重入次数
* 可重试：利用信号量和PubSub功能实现等待、唤醒，获取锁失败的重试机制
* 超时续约：利用watchDog，每隔一段时间（releaseTime /3），重置超时时间

#### 主从一致性问题：

![image-20220524172750043](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524172750043.png)

![image-20220524172842956](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524172842956.png)

**搭建3个redis节点:** 

只需要修改原来节点的端口号再启动 2个redis服务

![image-20220524173634837](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524173634837.png)

```java
    @Resource
    private RedissonClient redissonClient;
    @Resource
    private RedissonClient redissonClient2;
    @Resource
    private RedissonClient redissonClient3;

    private RLock lock;

    @BeforeEach
    void setUp(){
        RLock lock1 = redissonClient.getLock("order");
        RLock lock2 = redissonClient2.getLock("order");
        RLock lock3 = redissonClient3.getLock("order");
        //创建连锁
        lock=redissonClient.getMultiLock(lock1,lock2,lock3);

    }
```

1. 不可重入Redis分布式锁：
   * 原理：利用setnx的互斥性；利用ex避免死锁；释放锁时判断线程标示
   * 缺陷：不可重入、无法重试、锁超时失效

2. 可重入的Redis分布式锁：
   * 原理：利用hash结构，记录线程标示和重入次数；利用watchDog延续锁时间；利用信号量控制锁重试等待
   * 缺陷：redis宕机引起锁失效问题
3. Redisson的multiLock：
   * 原理：多个独立的Redis节点，必须在所有节点都获取重入锁，才算获取锁成功
   * 缺陷：运维成本高、实现复杂

## Redis优化秒杀业务

![image-20220524192634929](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524192634929.png)

对数据库的添加和更新操作我们开启新线程 ，抢不抢的到的资格问题我们把他交给redis来做 使用lua脚本。

redis检验的资格，就相当于我们去吃饭时的小票。

![image-20220524192723175](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524192723175.png)



![image-20220524193505920](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220524193505920.png)

**改进秒杀业务，提高并发性能**
需求：
① 新增秒杀优惠券的同时，将优惠券信息保存到Redis中

```java
@Override
    @Transactional
    public void addSeckillVoucher(Voucher voucher) {
        // 保存优惠券
        save(voucher);
        // 保存秒杀信息
        SeckillVoucher seckillVoucher = new SeckillVoucher();
        seckillVoucher.setVoucherId(voucher.getId());
        seckillVoucher.setStock(voucher.getStock());
        seckillVoucher.setBeginTime(voucher.getBeginTime());
        seckillVoucher.setEndTime(voucher.getEndTime());
        seckillVoucherService.save(seckillVoucher);
        //保存秒杀的库存 加入redis
        stringRedisTemplate.opsForValue().set(SECKILL_STOCK_KEY+voucher.getId(),voucher.getStock().toString());
    }
```

② 基于Lua脚本，判断秒杀库存、一人一单，决定用户是否抢购成功

```lua
-- 1.参数列表
--1.1 优惠卷id
local voucherId=ARGV[1]
-- 1.2 用户id
local userId=ARGV[2]

-- 2.数据key
-- 2.1 库存key
local stockKey='seckill:stock' .. voucherId
-- 2.2 订单key
local orderKey='seckill:order'.. voucherId
-- 3.脚本业务
-- 3.1 判断库存是否充足
-- 这里得到的时 string类型 不能直接判断
if(tonumber(redis.call('get',stockKey))<= 0) then
    -- 3.1.1 库存不足
    return 1;
end
-- 3.2 判断用户是否下单
if(redis.call('SISMEMBER',orderKey,userId)== 1) then
    -- 3.3存在，说明时重复下单吗，返回2
    return 2;
end
-- 3.4 扣库存 incrby stockKey -1
redis.call('incrby',stockKey,-1)
-- 3.5 下单(保存用户) sadd orderKey userId
redis.call('sadd',orderKey,userId)
return 0
```

③ 如果抢购成功，将优惠券id和用户id封装后存入阻塞队列

④ 开启线程任务，不断从阻塞队列中获取信息，实现异步下单功能

```java
 private  static final DefaultRedisScript<Long> SECKILL_SCRIPT;
  static {
        SECKILL_SCRIPT=new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("seckill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }
    private BlockingQueue<VoucherOrder> orderTasks=new ArrayBlockingQueue<>(1024*1024);
    private static final ExecutorService SECKILL_ORDER_EXECUTOR= Executors.newSingleThreadExecutor();
    private  IVoucherOrderService proxy;

   //类初始化完 开启线程池
    @PostConstruct
    private void init(){
        SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
    }
//秒杀业务开始 执行lua脚本 判断用户是否持有秒杀资格
    @Override
    public Result seckillVoucher(Long voucherId) {
        Long userId = UserHolder.getUser().getId();
        //1 .执行 lua脚本
        Long result = stringRedisTemplate.execute(SECKILL_SCRIPT, Collections.emptyList(), voucherId.toString(), userId.toString());
        //result 0:成功 1:库存不足 2:已经下单
        // 2.判断结果为0
        int r = result.intValue();
        if (r != 0){
            // 2.1. 不为0代表没有购买资格
            return Result.fail(r==1?"库存不足！":"您已经下单成功!");
        }
       // 2.2 为0 有购买资格，把下单信息保存到阻塞队列
        //生成订单号
        long orderId = redisWorker.nextId("order");
        //7 创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //订单id
        voucherOrder.setId(orderId);
        //用户id
        voucherOrder.setUserId(userId);
        //代金卷id
        voucherOrder.setVoucherId(voucherId);
        //3. 返回订单id
        orderTasks.add(voucherOrder);
       return  Result.ok(orderId);
    }
	//异步创建订单任务 再类初始化之后交给线程池执行
    private class VoucherOrderHandler implements  Runnable{
        @Override
        public void run() {
            try {
                while (true){
                    //1 获取队列中的订单信息
                    VoucherOrder voucherOrder = orderTasks.take();
                    //创建订单
                    handleVoucherOrder(voucherOrder);
                }
            } catch (Exception e) {
                log.error("处理订单异常!!",e);
            }
        }
    }
//创建订单和减库存
    private void handleVoucherOrder(VoucherOrder voucherOrder) {
        //为什么这里 还要做加锁呢 我们redis已经做了资格判断 不是不需要锁吗？
        //这是为了 以防万一 做兜底方案 防止redis宕机
        Long userId = voucherOrder.getUserId();
        Long voucherId = voucherOrder.getVoucherId();
        RLock lock = redissonClient.getLock("lock:order:" + userId);
        boolean isLock = lock.tryLock();
        //判断是否获取锁成功
        if (!isLock){
            //获取锁失败, 返回错误或重试
            log.error("不允许重复下单!");
            return;
        }
        try {
            proxy.createVoucherOrder(voucherOrder);
        } finally {
            lock.unlock();
        }
    }
    @Transactional
    public  void createVoucherOrder(VoucherOrder voucherOrder){
        Long userId = voucherOrder.getUserId();
        Long voucherId = voucherOrder.getVoucherId();
        int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
            //5.2 判断是否存在
            if (count > 0) {
                //用户已经购买
                log.error("用户已经购买!!");
                return;
            }
            //6 扣减库存
            boolean success = seckillVoucherService.update()
                    .setSql("stock=stock-1").eq("voucher_id", voucherId).gt("stock", 0) //只用判断库存大于0就可以了
                    .update();
            if (!success) {
                //扣减失败
                log.error("库存不足!!");
                return;
            }
            this.save(voucherOrder);
        }
```

**测试前置要求redis中一定储存秒杀卷的库存信息**

秒杀业务的优化思路是什么？
① 先利用Redis完成库存余量、一人一单判断，完成抢单业务
② 再将下单业务放入阻塞队列，利用独立线程异步下单
基于阻塞队列的异步秒杀存在哪些问题？

* 内存限制问题
* 数据安全问题



## Redis消息队列实现异步秒杀

**市面上还有一些其他的 消息队列如： Rabbitmq，RocketMq等**

消息队列（Message Queue），字面意思就是存放消息的队列。最简单的消息队列模型包括3个角色：

* 消息队列：存储和管理消息，也被称为消息代理（Message Broker）
* 生产者：发送消息到消息队列
* 消费者：从消息队列获取消息并处理消息

Redis提供了三种不同的方式来实现消息队列：

* list结构：基于List结构模拟消息队列
* PubSub：基本的点对点消息模型
* Stream：比较完善的消息队列模型

![image-20220525144650380](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525144650380.png)



#### 基于**List实现消息队列**

消息队列（Message Queue），字面意思就是存放消息的队列。而Redis的list数据结构是一个双向链表，很容易模拟出队列效果。
队列是入口和出口不在一边，因此我们可以利用：LPUSH 结合 RPOP、或者 RPUSH 结合 LPOP来实现。
不过要注意的是，当队列中没有消息时RPOP或LPOP操作会返回null，并不像JVM的阻塞队列那样会阻塞并等待消息。因此这里应该使用BRPOP或者BLPOP来实现阻塞效果。

![image-20220525151831375](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525151831375.png)

基于List的消息队列有哪些优缺点？
优点：

* 利用Redis存储，不受限于JVM内存上限
* 基于Redis的持久化机制，数据安全性有保证
* 可以满足消息有序性

缺点：

* 无法避免消息丢失
* 只支持单消费者

#### 基于PubSub的消息队列

PubSub（发布订阅）是Redis2.0版本引入的消息传递模型。顾名思义，消费者可以订阅一个或多个channel，生产者向对应channel发送消息后，所有订阅者都能收到相关消息。

* SUBSCRIBE channel [channel] ：订阅一个或多个频道
* PUBLISH channel msg ：向一个频道发送消息
* PSUBSCRIBE pattern[pattern] ：订阅与pattern格式匹配的所有频道

![image-20220525152638028](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525152638028.png)

基于PubSub的消息队列有哪些优缺点？
优点：

* 采用发布订阅模型，支持多生产、多消费

缺点：

* 不支持数据持久化
* 无法避免消息丢失
* 消息堆积有上限，超出时数据丢失

#### 基于Stream的消息队列

Stream 是 Redis 5.0 引入的一种新数据类型，可以实现一个功能非常完善的消息队列。
发送消息的命令：

![image-20220525154254793](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525154254793.png)

例如：

![image-20220525154306147](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525154306147.png)

读取消息的方式之一：**XREAD**

![image-20220525154835352](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525154835352.png)

例如，使用XREAD读取第一个消息：

![image-20220525154826239](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525154826239.png)阻塞读取消息:

![image-20220525155512816](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525155512816.png)

**0表示一致阻塞 直到有消息**

STREAM类型消息队列的XREAD命令特点：

* 消息可回溯
*  一个消息可以被多个消费者读取
* 可以阻塞读取
* 有消息漏读的风险

消费者组（Consumer Group）：将多个消费者划分到一个组中，监听同一个队列。具备下列特点：

![image-20220525160058405](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525160058405.png)

创建消费者组：

```redis
XGROUP CREATE key groupName ID [MKSTREAM]
```

* key：队列名称
* groupName：消费者组名称
* ID：起始ID标示，$代表队列中最后一个消息，0则代表队列中第一个消息
* MKSTREAM：队列不存在时自动创建队列

其它常见命令：

```shell
# 删除指定的消费者组
XGROUP DESTORY key groupName
# 给指定的消费者组添加消费者
XGROUP CREATECONSUMER key groupname consumername
# 删除消费者组中的指定消费者
XGROUP DELCONSUMER key groupname consumername
```

从消费者组读取消息：

```shell
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID
[ID ...]
```

group：消费组名称

* consumer：消费者名称，如果消费者不存在，会自动创建一个消费者
* count：本次查询的最大数量
* BLOCK milliseconds：当没有消息时最长等待时间
* NOACK：无需手动ACK，获取到消息后自动确认
* STREAMS key：指定队列名称
* ID：获取消息的起始ID：
  * ">"：从下一个未消费的消息开始
  * 其它：根据指定id从pending-list中获取已消费但未确认的消息，例如0，是从pending-list中的第一个消息开始

```shell
#key:队列名称 group:组名 IDLE min-idle-time：空闲时间，空闲时间超过这个的消息 start:开始位置 end:结束位置 count:数量 consumer:那个消费者
XPENDING key group [ [IDLE min-idle-time] start end count [consumer]]
```

例如:

```shell
#表示获取未确认的消息
xpending s1 g1 - + 10
```

![image-20220525162953985](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525162953985.png)

STREAM类型消息队列的XREADGROUP命令特点：

* 消息可回溯
* 可以多消费者争抢消息，加快消费速度
* 可以阻塞读取
* 没有消息漏读的风险
* 有消息确认机制，保证消息至少被消费一次

![image-20220525163254215](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525163254215.png)



案例：

需求：
① 创建一个Stream类型的消息队列，名为stream.orders

```
xgroup create stream.orders g1 0 MKstream
```

② 修改之前的秒杀下单Lua脚本，在认定有抢购资格后，直接向stream.orders中添加消息，内容包含voucherId、userId、orderId
③ 项目启动时，开启一个线程任务，尝试获取stream.orders中的消息，完成下单

```java
 private static final ExecutorService SECKILL_ORDER_EXECUTOR= Executors.newSingleThreadExecutor();
    private  IVoucherOrderService proxy;
    @PostConstruct
    private void init(){
        SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
    }
    private class VoucherOrderHandler implements  Runnable{
        String queueName="stream.orders";
        @Override
        public void run() {
            try {
                while (true){
                    //1 获取队列中的订单信息 XREADGROUP GROUP g1 c1 count 1 block 2000 streams.order >
                    List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                            Consumer.from("g1", "c1"),
                            StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                            StreamOffset.create(queueName, ReadOffset.lastConsumed()));
                    //2 判断消息是否获取成功
                    if (list==null || list.isEmpty()){
                        //2.1 如果获取失败，说明没有消息，继续下一次循环
                        continue;
                    }
                    MapRecord<String, Object, Object> map = list.get(0);
                    Map<Object, Object> values = map.getValue();
                    VoucherOrder voucherOrder = new VoucherOrder();
                    BeanUtil.fillBeanWithMap(values,voucherOrder,true);
                    //3.如果获取成功，可以下单
                    handleVoucherOrder(voucherOrder);
                    //4 ACK 确认 SACK stream.orders g1 id
                    stringRedisTemplate.opsForStream().acknowledge(queueName,"g1",map.getId());
                }
            } catch (Exception e) {
                log.error("处理订单异常!!",e);
                handlePendingList();
            }
        }
        //创建订单失败处理流程
        private void handlePendingList() {
            while (true){
                try {
                    List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                            Consumer.from("g1", "c1"),
                            StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                            StreamOffset.create(queueName, ReadOffset.from("0")));
                    if (list==null|| list.isEmpty()){
                        //如果获取失败,说明pending-list没有异常信息,继续下一次循环
                        break;
                    }
                    MapRecord<String, Object, Object> map = list.get(0);
                    Map<Object, Object> values = map.getValue();
                    VoucherOrder voucherOrder = new VoucherOrder();
                    BeanUtil.fillBeanWithMap(values,voucherOrder,true);
                    //3.如果获取成功，可以下单
                    handleVoucherOrder(voucherOrder);
                    //4 ACK 确认 SACK stream.orders g1 id
                    stringRedisTemplate.opsForStream().acknowledge(queueName,"g1",map.getId());
                } catch (Exception e) {
                    log.error("处理订单异常!!",e);
                    try {
                        Thread.sleep(20);
                    } catch (InterruptedException interruptedException) {
                        interruptedException.printStackTrace();
                    }
                }
            }
        }
    }
    private void handleVoucherOrder(VoucherOrder voucherOrder) {

        //为什么这里 还要做加锁呢 我们redis已经做了资格判断 不是不需要锁吗？
        //这是为了 以防万一 做兜底方案 防止redis宕机
        Long userId = voucherOrder.getUserId();
        Long voucherId = voucherOrder.getVoucherId();
        RLock lock = redissonClient.getLock("lock:order:" + userId);
        boolean isLock = lock.tryLock();
        //判断是否获取锁成功
        if (!isLock){
            //获取锁失败, 返回错误或重试
            log.error("不允许重复下单!");
            return;
        }
        try {
            proxy.createVoucherOrder(voucherOrder);
        } finally {
            lock.unlock();
        }
    }
    @Transactional
    public  void  createVoucherOrder(VoucherOrder voucherOrder){
        Long userId = voucherOrder.getUserId();
        Long voucherId = voucherOrder.getVoucherId();
        int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
            //5.2 判断是否存在
            if (count > 0) {
                //用户已经购买
                log.error("用户已经购买!!");
                return;
            }
            //6 扣减库存
            boolean success = seckillVoucherService.update()
                    .setSql("stock=stock-1").eq("voucher_id", voucherId).gt("stock", 0) //只用判断库存大于0就可以了
                    .update();
            if (!success) {
                //扣减失败
                log.error("库存不足!!");
                return;
            }
            this.save(voucherOrder);
        }
    @Override
    public Result seckillVoucher(Long voucherId) {
        Long userId = UserHolder.getUser().getId();
        long orderId = redisWorker.nextId("order");
        //1 .执行 lua脚本
        Long result = stringRedisTemplate.execute(SECKILL_SCRIPT, Collections.emptyList(), voucherId.toString(), userId.toString(),String.valueOf(orderId));
        //result 0:成功 1:库存不足 2:已经下单
        // 2.判断结果为0
        int r = result.intValue();
        if (r != 0){
            // 2.1. 不为0代表没有购买资格
            return Result.fail(r==1?"库存不足！":"您已经下单成功!");
        }
        proxy = (IVoucherOrderService)AopContext.currentProxy();
        return  Result.ok(orderId);
    }
```



