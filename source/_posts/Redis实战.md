---
title: Redis实战1
date: 2022-05-16 23:33:07
categories: Redis
---

# 短信登录

导入sql文件:

 tb_user：用户表
 tb_user_info：用户详情表
 tb_shop：商户信息表
 tb_shop_type：商户类型表
 tb_blog：用户日记表（达人探店日记）
 tb_follow：用户关注表
 tb_voucher：优惠券表
 tb_voucher_order：优惠券的订单表

![image-20220517152907974](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220517152907974.png)

## 基于Session实现登录

![image-20220517154554668](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220517154554668.png)

**发送验证码核心代码:**

```java
@Slf4j
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {
    @Override
    public Result sendCode(String phone, HttpSession session) {

        //1.校验手机号
        if (RegexUtils.isPhoneInvalid(phone)){
            //2 如果不符合，返回错误信息
            return Result.fail("手机号格式错误！！");
        }
        //3 符合 生产验证码
        String code = RandomUtil.randomNumbers(6);
        //4 保存验证码到session
        session.setAttribute("code",code);
        //5 发送验证码
        log.debug("发送验证码成功");
        // 返回ok
        return Result.ok();
    }
}
```

 **验证码登录，注册**

```java
 @Override
    public Result login(LoginFormDTO loginForm, HttpSession session) {
        //1 校验手机号
        String phone=loginForm.getPhone();
        if(RegexUtils.isPhoneInvalid(phone)){
            return  Result.fail("手机号格式错误");
        }
        //2. 校验验证码
        Object cacheCode = session.getAttribute("code");
        //3 不一致 报错
        String code = loginForm.getCode();
        if (cacheCode==null || !cacheCode.toString().equals(code))
        {
            return Result.fail("验证码错误！！");
        }
        //4 一致 根据手机号查询用户
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper.eq("phone",phone);
        User user = baseMapper.selectOne(wrapper);
        //5 判断用户是否存在
        if (user==null){
            //6 不存在 保存在数据库
            user= createUserWithPhone(phone);
        }
        //7 保存用户到session
        session.setAttribute("user",user);
        return Result.ok();
    }

    private User createUserWithPhone(String phone) {
        User user = new User();
        user.setPhone(phone);
        user.setNickName(USER_NICK_NAME_PREFIX+RandomUtil.randomString(10));
        baseMapper.insert(user);
        return user;
    }
```

**登录验证功能**

配置拦截器:

```java
public class LoginInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //1 获取session
        HttpSession session = request.getSession();
        //2 获取session中的用户
        User user =(User) session.getAttribute("user");
        //3 判断用户是否存在
        if (user==null){
            //4 不存在 拦截 返回401状态码
            response.setStatus(401);
            return false;
        }
        //5 存在保存用户信息在ThreadLocal
        UserDTO userDTO = new UserDTO();
        userDTO.setId(user.getId());
        userDTO.setNickName(user.getNickName());
        userDTO.setIcon(user.getIcon());
        UserHolder.saveUser(userDTO);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

        //移除用户 防止内存泄漏
        UserHolder.removeUser();
    }
}

```

添加拦截器:

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                .excludePathPatterns(
                        "/user/login",
                        "/user/code",
                        "/blog/hot",
                        "/shop/**",
                        "/shop-type/**",
                        "/voucher/**"
                        );
    }
}
```

## 集群的session共享问题

session共享问题：多台Tomcat并不共享session存储空间，当请求切换到不同tomcat服务时导致数据丢失的问题。
session的替代方案应该满足：

* 数据共享
* 内存存储
* key、value结构 

![image-20220517171925508](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220517171925508.png)

redis秀修改发送短信验证码:

```java
@Override
    public Result sendCode(String phone, HttpSession session) {
        //1.校验手机号
        if (RegexUtils.isPhoneInvalid(phone)){
            //2 如果不符合，返回错误信息
            return Result.fail("手机号格式错误！！");
        }
        //3 符合 生产验证码
        String code = RandomUtil.randomNumbers(6);
        //4 保存验证码到redis
        stringRedisTemplate.opsForValue().set(LOGIN_CODE_KEY+phone,code,LOGIN_CODE_TTL, TimeUnit.MINUTES);
        //5 发送验证码
        log.debug("发送验证码成功!!为:{}",code);
        // 返回ok
        return Result.ok();
    }
```

短信验证码登录，注册

```java
@Override
    public Result login(LoginFormDTO loginForm, HttpSession session) {
        //1 校验手机号
        String phone=loginForm.getPhone();
        if(RegexUtils.isPhoneInvalid(phone)){
            return  Result.fail("手机号格式错误");
        }
        //2. 校验验证码
        String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);
        // 3 不一致 报错
        String code = loginForm.getCode();
        if (cacheCode==null || !cacheCode.toString().equals(code))
        {
            return Result.fail("验证码错误！！");
        }
        //4 一致 根据手机号查询用户
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper.eq("phone",phone);
        User user = baseMapper.selectOne(wrapper);
        //5 判断用户是否存在
        if (user==null){
            //6 不存在 保存在数据库
            user= createUserWithPhone(phone);
        }
        //7 保存用户到session
        //7.1 随机生成token，作为登陆令牌
        String token = UUID.randomUUID().toString(true);
        //7.2 将User对象转为Hash存储
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        Map<String, Object> map = BeanUtil.beanToMap(userDTO);
        //存储
        String tokenKey = LOGIN_USER_KEY + token;
        stringRedisTemplate.opsForHash().putAll(tokenKey,map);
        stringRedisTemplate.expire(tokenKey,LOGIN_USER_TTL,TimeUnit.MINUTES);
        return Result.ok();
    }

    private User createUserWithPhone(String phone) {
        User user = new User();
        user.setPhone(phone);
        user.setNickName(USER_NICK_NAME_PREFIX+RandomUtil.randomString(10));
        baseMapper.insert(user);
        return user;
    }
```

拦截器修改 更新用户信息有效期

```java
public class LoginInterceptor implements HandlerInterceptor {

    private StringRedisTemplate stringRedisTemplate;

    public LoginInterceptor(StringRedisTemplate redisTemplate){
        this.stringRedisTemplate=redisTemplate;
    }
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //1 获取请求头中的token

        String token = request.getHeader("authorization");
        //2 基于token获取redis中的用户
        if (StrUtil.isBlank(token)) {
            response.setStatus(401);
            return false;
        }
        String key = RedisConstants.LOGIN_USER_KEY + token;
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
        //3 判断用户是否存在
        if (userMap.isEmpty()){
            //4 不存在 拦截 返回401状态码
            response.setStatus(401);
            return false;
        }
        //5 存在保存用户信息在ThreadLocal
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        //存储
        UserHolder.saveUser(userDTO);
        //刷新token有效期
        stringRedisTemplate.expire(key,RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //移除用户 防止内存泄漏
        UserHolder.removeUser();
    }
}

@Configuration
public class MvcConfig implements WebMvcConfigurer {

    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor(stringRedisTemplate))
                .excludePathPatterns(
                        "/user/login",
                        "/user/code",
                        "/blog/hot",
                        "/shop/**",
                        "/shop-type/**",
                        "/voucher/**"
                        );
    }
}

```

异常出现: 

StringRedisTemplate 存储的时候要保证map 对应的 key和value都是String才行 我们id 是Long类型所以出现类型转换失败

![image-20220517220231650](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220517220231650.png)

修改service层 登录验证代码

```java
 	@Override
    public Result login(LoginFormDTO loginForm, HttpSession session) {
        //1 校验手机号
        String phone=loginForm.getPhone();
        if(RegexUtils.isPhoneInvalid(phone)){
            return  Result.fail("手机号格式错误");
        }
        //2. 校验验证码
        String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);
        // 3 不一致 报错
        String code = loginForm.getCode();
        if (cacheCode==null || !cacheCode.toString().equals(code))
        {
            return Result.fail("验证码错误！！");
        }
        //4 一致 根据手机号查询用户
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper.eq("phone",phone);
        User user = baseMapper.selectOne(wrapper);
        //5 判断用户是否存在
        if (user==null){
            //6 不存在 保存在数据库
            user= createUserWithPhone(phone);
        }
        //7 保存用户到session
        //7.1 随机生成token，作为登陆令牌
        String token = UUID.randomUUID().toString(true);
        //7.2 将User对象转为Hash存储
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        Map<String, Object> map = BeanUtil.beanToMap(userDTO,new HashMap<>(),CopyOptions.create().setIgnoreNullValue(true).setFieldValueEditor((fieldName,fieldValue)->fieldValue.toString()));
        //存储
        String tokenKey = LOGIN_USER_KEY + token;
        stringRedisTemplate.opsForHash().putAll(tokenKey,map);
        stringRedisTemplate.expire(tokenKey,LOGIN_USER_TTL,TimeUnit.MINUTES);
        //返回token给前端 然后通过 拦截器 拦截请求头中token信息 查询 redis 存储到ThreadLocal中 
        return Result.ok(token);
    }
  private User createUserWithPhone(String phone) {
        User user = new User();
        user.setPhone(phone);
        user.setNickName(USER_NICK_NAME_PREFIX+RandomUtil.randomString(10));
        baseMapper.insert(user);
        return user;
    }
```

Redis代替session需要考虑的问题：

* 选择合适的数据结构
* 选择合适的key
*  选择合适的存储粒度 

![image-20220517230725671](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220517230725671.png)

![image-20220517230737656](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220517230737656.png)

要控制拦截器的顺序： RefreshTokenInterceptor->LoginInterceptor

RefreshTokenInterceptor:

```java
public class RefreshTokenInterceptor implements HandlerInterceptor {

    private StringRedisTemplate stringRedisTemplate;

    public RefreshTokenInterceptor(StringRedisTemplate redisTemplate){
        this.stringRedisTemplate=redisTemplate;
    }
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //1 获取请求头中的token
        String token = request.getHeader("authorization");
        if (StrUtil.isBlank(token)) {
            return true;
        }
        String key = RedisConstants.LOGIN_USER_KEY + token;
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
        //3 判断用户是否存在
        if (userMap.isEmpty()){
            return true;
        }
        //5 存在保存用户信息在ThreadLocal
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        //存储
        UserHolder.saveUser(userDTO);
        //刷新token有效期
        stringRedisTemplate.expire(key,RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);
        return true;
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //移除用户 防止内存泄漏
        UserHolder.removeUser();
    }
}
```

LoginInterceptor

```java
public class LoginInterceptor implements HandlerInterceptor {
    private StringRedisTemplate stringRedisTemplate;
    public LoginInterceptor(StringRedisTemplate redisTemplate){
        this.stringRedisTemplate=redisTemplate;
    }
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //1 判断是否需要拦截(ThreadLocal是是否头用户)
        if (UserHolder.getUser()==null){
            response.setStatus(401);
            return false;
        }
        //由用户 ，则方行
        return true;
    }
}
```

MvcConfig:

```java
//order 控制拦截器执行顺序 越小越先执行
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                .excludePathPatterns(
                        "/user/login",
                        "/user/code",
                        "/blog/hot",
                        "/shop/**",
                        "/shop-type/**",
                        "/voucher/**"
                        ).order(1);
        registry.addInterceptor(new RefreshTokenInterceptor(stringRedisTemplate)).order(0);
    }
}
```

#  商户查询缓存

## 什么是缓存

缓存就是数据交换的缓冲区（称作Cache [ kæʃ ] ），是存贮数据的临时地方，一般读写性能较高

缓存的作用:

* 降低后端负载
* 提高读写效率，降低响应时间

缓存的成本:

* 数据一致性成本
* 代码维护成本
* 运维成本

![image-20220518203300651](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518203300651.png)

![image-20220518203649821](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518203649821.png)

给商铺详情信息：添加缓存

 ```java
 @Service
 public class ShopServiceImpl extends ServiceImpl<ShopMapper, Shop> implements IShopService {
 
     @Autowired
     private StringRedisTemplate redisTemplate;
 
     @Override
     public Result queryById(Long id) {
 
         String key = CACHE_SHOP_KEY + id;
         //1 从redis查询商铺缓存
 //        Map<Object, Object> map = redisTemplate.opsForHash().entries(key);
         String shopJson = redisTemplate.opsForValue().get(key);
         //2 判断是否存在
 
         if (StrUtil.isNotBlank(shopJson)){
             //3 存在，直接返回
 //            Shop shop = BeanUtil.fillBeanWithMap(map, new Shop(), false);
             Shop shop = JSONUtil.toBean(shopJson, Shop.class);
             return Result.ok(shop);
         }
         //4不存在，根据id查询数据库
         Shop shop = getById(id);
         //5 不存在返回错误
         if (shop==null){
             return Result.fail("店铺不存在");
         }
         //6 存在，写入redis
 //        Map<String, Object> map1 = BeanUtil.beanToMap(shop,new HashMap<>(), CopyOptions.create().setIgnoreNullValue(true).setFieldValueEditor(((fieldKey,fieldValue)->fieldValue.toString())));
 //        redisTemplate.opsForHash().putAll(key,map1);
         String jsonStr = JSONUtil.toJsonStr(shop);
         redisTemplate.opsForValue().set(key,jsonStr);
         return Result.ok(shop);
     }
 }
 ```

List缓存 商店分类数据 :

```java
@Service
public class ShopTypeServiceImpl extends ServiceImpl<ShopTypeMapper, ShopType> implements IShopTypeService {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public Result queryAllType() {
        //1 查询redis缓存

        //String存储
//        String shopTypeJson = stringRedisTemplate.opsForValue().get(CACHE_SHOP_TYPE_KEY);
        //List存储
        List<String> range = stringRedisTemplate.opsForList().range(CACHE_SHOP_TYPE_KEY, 0L, -1L);
        //2 是否存在redis缓存
        if (range!=null&&!range.isEmpty()){
            // 3存在redis缓存
            ArrayList<ShopType> shopTypes = new ArrayList<>();
            range.forEach((stringShopType)-> shopTypes.add(JSONUtil.toBean(stringShopType,ShopType.class))  );
            return Result.ok(shopTypes);
        }
        //4 不存在redis 缓存查询数据库
        List<ShopType> shopTypes = query().orderByAsc("sort").list();
        //5 没有类型返回失败
        if (shopTypes==null ||shopTypes.isEmpty())
        {
            return Result.fail("没有商铺类型");
        }
        ArrayList<String> stringShopTypes = new ArrayList<>();
        //把List 元素 ShopType 全部转 String类型
        shopTypes.forEach(shopType -> stringShopTypes.add(JSONUtil.toJsonStr(shopType)));
        
        //6 存储到redis中
        String toJsonStr = JSONUtil.toJsonStr(shopTypes);
        
        //String存储
//        stringRedisTemplate.opsForValue().set(CACHE_SHOP_TYPE_KEY,toJsonStr);
        //List存储
        stringRedisTemplate.opsForList().rightPushAll(CACHE_SHOP_TYPE_KEY,stringShopTypes);
        return Result.ok(shopTypes);
    }
}
```

## 缓存更新策略

业务场景：

* 低一致性需求：**使用内存 淘汰机制 或过期淘汰。例如店铺类型的查询缓存**
* 高一致性需求：**主动更新，并以超时剔除作为兜底方案。例如店铺详情查询的缓存**

![image-20220518220436618](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518220436618.png)

操作缓存和数据库时有三个问题需要考虑：
1. 删除缓存还是更新缓存？
  * 更新缓存：每次更新数据库都更新缓存，无效写操作 较多
  * 删除缓存：更新数据库时让缓存失效，查询时再更新缓存
2. 如何保证缓存与数据库的操作的同时成功或失败？
  * 单体系统，将缓存与数据库操作放在一个事务
  * 分布式系统，利用TCC等分布式事务方案
3. 先操作缓存还是先操作数据库？
* 先删除缓存，再操作数据库
* 先操作数据库，再删除缓存
* ![image-20220518221539995](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518221539995.png)

​	对比方案2 的缓存不一致问题几率更低

给查询商铺的缓存添加超时剔除和主动更新的策略

```java
@Override
    @Transactional
    public Result update(Shop shop) {
        Long id = shop.getId();
        if (id==null) {
            return Result.fail("店铺id不能为空");
        }
        //1 更新数据库
        updateById(shop);
        //2 删除缓存
        redisTemplate.delete(CACHE_SHOP_KEY+id);
        return Result.ok();
    }
```

## 缓存穿透

缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库。

常见的解决方案有两种：

* 缓存空对象
* 优点：实现简单，维护方便 用户id作为key
* 缺点：
  * 额外的内存消耗
  * 可能造成短期的不一致

![image-20220518224426526](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518224426526.png)

```java
 @Override
    public Result queryById(Long id) {

        String key = CACHE_SHOP_KEY + id;
        //1 从redis查询商铺缓存
//        Map<Object, Object> map = redisTemplate.opsForHash().entries(key);
        String shopJson = redisTemplate.opsForValue().get(key);
        //2 判断是否存在
        if (StrUtil.isNotBlank(shopJson)){
            //3 存在，直接返回
//            Shop shop = BeanUtil.fillBeanWithMap(map, new Shop(), false);
            Shop shop = JSONUtil.toBean(shopJson, Shop.class);
            return Result.ok(shop);
        }
        //判断 数据为空值  解决缓存穿透 到数据库
        if (shopJson!=null){
            return  Result.fail("店铺信息不存在!");
        }
        //4不存在，根据id查询数据库
        Shop shop = getById(id);
        //5 不存在返回错误
        if (shop==null){
            //将空值写入 redis 解决缓存击穿
            redisTemplate.opsForValue().set(key,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
            return Result.fail("店铺不存在");
        }
        //6 存在，写入redis
//        Map<String, Object> map1 = BeanUtil.beanToMap(shop,new HashMap<>(), CopyOptions.create().setIgnoreNullValue(true).setFieldValueEditor(((fieldKey,fieldValue)->fieldValue.toString())));
//        redisTemplate.opsForHash().putAll(key,map1);
        String jsonStr = JSONUtil.toJsonStr(shop);
        redisTemplate.opsForValue().set(key,jsonStr,CACHE_SHOP_TTL, TimeUnit.MINUTES);
        return Result.ok(shop);
    }
```



* 布隆过滤
* 优点：内存占用较少，没有多余key
* 缺点：
  * 实现复杂
  * 存在误判可能

![image-20220518224439789](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518224439789.png)

缓存穿透产生的原因是什么？

* 用户请求的数据在缓存中和数据库中都不存在，不断发起这样的请求，给数据库带来巨大压力

缓存穿透的解决方案有哪些？

* 缓存null值
*  布隆过滤
* 增强id的复杂度，避免被猜测id规律
* 做好数据的基础格式校验
* 加强用户权限校验
* 做好热点参数的限流

## 缓存雪崩

缓存雪崩是指在同一时段大量的缓存key同时失效或者Redis服务宕机，导致大量请求到达数据库，带来巨大压力。

解决方案：

* 给不同的Key的TTL添加随机值
* 利用Redis集群提高服务的可用性
* 给缓存业务添加降级限流策略 利用Springcloud做服务降级
* 给业务添加多级缓存

![image-20220518231322492](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518231322492.png)

##  缓存击穿

缓存击穿问题也叫热点Key问题，就是一个被**高并发访问**并且**缓存重建业务较复杂的key**突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击。

解决方案：

* 互斥锁

* 逻辑过期

互斥锁:

![image-20220518232634140](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518232634140.png)

互斥锁实现:

![image-20220518234446548](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518234446548.png)

```java
public Shop queryWhitMutex(Long id) {
        String key = CACHE_SHOP_KEY + id;
        String shopJson = redisTemplate.opsForValue().get(key);
        //2 判断是否存在
        if (StrUtil.isNotBlank(shopJson)){
            return JSONUtil.toBean(shopJson, Shop.class);
        }
        //判断 数据为空值  解决缓存穿透 到数据库
        if (shopJson!=null){
            return  null;
        }
        //4 实现缓存重建
        //4.1 获取互斥锁
        String lockKey = null;
        Shop shop = null;
        try {
            lockKey = LOCK_SHOP_KEY + id;
            boolean isLock = tryLock(lockKey);
            //4.2 判断是否获取成功
            if (!isLock) {
                //4.3 失败，休眠并重试
                try {
                    Thread.sleep(50);
                   return queryWhitMutex(id);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            //4.4 成功 根据id查询数据库
            //4，根据id查询数据库
            shop = getById(id);
            //模拟重建的延时
            Thread.sleep(200);
            //5 不存在返回错误
            if (shop==null){
                //将空值写入 redis
                redisTemplate.opsForValue().set(key,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
                return null;
            }
            //6 存在，写入redis
            String jsonStr = JSONUtil.toJsonStr(shop);
            redisTemplate.opsForValue().set(key,jsonStr,CACHE_SHOP_TTL, TimeUnit.MINUTES);
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            //7 释放互斥锁
            unlock(lockKey);
        }
        return shop;
    }
```



逻辑过期:

![image-20220518232807204](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220518232807204.png)

![image-20220519200328591](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220519200328591.png)

```java
public Shop queryWhitLogicalExpire(Long id) {
        String key = CACHE_SHOP_KEY + id;
        String shopJson = redisTemplate.opsForValue().get(key);
        //2 判断是否存在
        if (StrUtil.isBlank(shopJson)){
            //不存在 返回店铺信息
            return null;
        }
        //4 命中 需要先把json反序列化为对象
        RedisData redisData = JSONUtil.toBean(shopJson, RedisData.class);
        JSONObject data = (JSONObject)redisData.getData();
        Shop shop = JSONUtil.toBean(data, Shop.class);
        LocalDateTime expireTime = redisData.getExpireTime();
        //5 判断是否过期
        if (expireTime.isAfter(LocalDateTime.now())){
            //5.1 过期时间在当前时间之后 未过期 直接返回店铺信息
            return shop;
        }
        //5.2 已过期，需要缓存重建
        //6缓存重建
        String lockKey= LOCK_SHOP_KEY+id;
        //6.1 获取互斥锁
        boolean isLock = tryLock(lockKey);
        //6.2 判断是否获取锁成功
        if (isLock){
            //6.3 成功,开启线程实现
            CACHE_REBUILD_EXECUTOR.submit(()->{
                //缓存重建
                try {
                    saveShop2Redis(id,20L);
                } catch (Exception e) {
                    throw  new RuntimeException(e);
                } finally {
                    unlock(lockKey);
                }
            });
        }
        //6.4 返回过期的店铺信息
        return shop;
    }

 public void saveShop2Redis(Long id,Long expireSeconds){
        //1 查询店铺数据
        Shop shop = getById(id);
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //2 封装逻辑过期时间
        RedisData redisData = new RedisData();
        redisData.setData(shop);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireSeconds));
        //3 写入redis
        redisTemplate.opsForValue().set(CACHE_SHOP_KEY+id,JSONUtil.toJsonStr(redisData));

    }
```

**互斥锁**

​	**优点**

		1. 没有额外的内存消耗
		1. 保证一致性
		1. 实现简单

​	**缺点**

1. 线程需要等待，性能受影响
2. 可能有死锁风险

**逻辑过期**

​	**优点**

1. 线程无需等待，性能较好

​	**缺点**

1. 不保证一致性
2. 有额外的内存消耗
3. 实现复杂

## 缓存工具封装:

基于StringRedisTemplate封装一个缓存工具类，满足下列需求：

* 方法1：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置TTL过期时间
* 方法2：将任意Java对象序列化为json并 存储在string类型的key中，并且可以设置逻辑过期时间，用于处理缓存击穿问题
*  方法3：根据指定的key查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题
* 方法4：根据指定的key查询缓存，并反序列化为指定类型，需要利用逻辑过期解决缓存击穿问题

```java
package com.hmdp.utils;
import cn.hutool.core.util.BooleanUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.hmdp.entity.Shop;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import static com.hmdp.utils.RedisConstants.*;
@Component
@Slf4j
public class CacheClient {
    private static final ExecutorService CACHE_REBUILD_EXECUTOR= Executors.newFixedThreadPool(10);


    private StringRedisTemplate stringRedisTemplate;

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }
    public void set(String key, Object value, Long time,TimeUnit unit){
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value),time,unit);
    }

    public void setWithLogicalExpire(String key,Object value,Long time,TimeUnit unit){
        //设置逻辑过期
        RedisData redisData=new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        stringRedisTemplate.opsForValue().set(key,JSONUtil.toJsonStr(redisData));
    }

    /**
     * 缓存穿透
     */
    public <R,ID> R queryWhitPassThrough(
            String keyPrefix, ID id, Class<R> type, Function<ID,R> dbFallback,Long time,TimeUnit unit) {
        String key = keyPrefix + id;
        String json = stringRedisTemplate.opsForValue().get(key);
        //2 判断是否存在
        if (StrUtil.isNotBlank(json)){
            //存在返回空
            return JSONUtil.toBean(json, type);
        }
        //判断 数据为空值  解决缓存穿透 到数据库
        if (json!=null){
            //缓存的空值 返回一个错误信息
            return  null;
        }
        //4不存在，根据id查询数据库 我们不知道怎么查
        R r = dbFallback.apply(id);
        //5 不存在返回错误
        if (r==null){
            //将空值写入 redis
            stringRedisTemplate.opsForValue().set(key,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
            return null;
        }
        //6 存在，写入redis
        this.set(key,r,time,unit);
        return r;
    }

    public <R,ID> R queryWhitLogicalExpire( String keyPrefix, ID id, Class<R> type, Function<ID,R> dbFallback,Long time,TimeUnit unit) {
        String key = keyPrefix + id;
        String json = stringRedisTemplate.opsForValue().get(key);
        //2 判断是否存在
        if (StrUtil.isBlank(json)){
            //不存在
            return null;
        }
        //4 命中 需要先把json反序列化为对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        JSONObject data = (JSONObject)redisData.getData();
        R r = JSONUtil.toBean(data, type);
        LocalDateTime expireTime = redisData.getExpireTime();
        //5 判断是否过期
        if (expireTime.isAfter(LocalDateTime.now())){
            //5.1 过期时间在当前时间之后 未过期 直接返回店铺信息
            return r;
        }
        //5.2 已过期，需要缓存重建
        //6缓存重建
        String lockKey= LOCK_SHOP_KEY+id;
        //6.1 获取互斥锁
        boolean isLock = tryLock(lockKey);
        //6.2 判断是否获取锁成功
        if (isLock){
            //6.3 成功,开启线程实现
            CACHE_REBUILD_EXECUTOR.submit(()->{
                //缓存重建
                try {
                   // 从数据库查询缓存
                    R r1 = dbFallback.apply(id);
                    //存入redis
                    this.setWithLogicalExpire(key,r1,time,unit);
                } catch (Exception e) {
                    throw  new RuntimeException(e);
                } finally {
                    unlock(lockKey);
                }
            });
        }
        //6.4 返回过期的店铺信息
        return r;
    }
    private boolean tryLock(String key){
        Boolean aBoolean = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", LOCK_SHOP_TTL, TimeUnit.SECONDS);

        return BooleanUtil.isTrue(aBoolean);
    }

    private  void unlock(String key){
        stringRedisTemplate.delete(key);
    }
}

//工具类使用
public Result queryById(Long id) {
        // 工具类实现缓存穿透
        //(id2)-> this.getById(id2) 对应下面简写
//        cacheClient.queryWhitPassThrough(CACHE_SHOP_KEY,id,Shop.class,this::getById,CACHE_SHOP_TTL,TimeUnit.MINUTES);
        //工具类 解决缓存击穿
        Shop shop = cacheClient.queryWhitLogicalExpire(CACHE_SHOP_KEY, id, Shop.class, this::getById, 20L, TimeUnit.SECONDS);

        if (shop==null){
            return Result.fail("店铺不存在");
        }
        return Result.ok(shop);
    }
```

