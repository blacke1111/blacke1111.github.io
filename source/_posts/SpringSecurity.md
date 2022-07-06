---
title: SpringSecurity
date: 2022-06-04 11:13:58
categories: SpringSecurity
---

## 权限管理中的相关概念

主体
英文单词：principal
使用系统的用户或设备或从其他系统远程登录的用户等等。简单说就是谁使用系统谁就是主体。
认证
英文单词：authentication
权限管理系统确认一个主体的身份，允许主体进入系统。简单说就是“主体”证明自己是谁。
笼统的认为就是以前所做的登录操作。
授权
英文单词：authorization
将操作系统的“权力”“授予”“主体”，这样主体就具备了操作系统中特定功能的能力。
所以简单来说，授权就是给用户分配权限。

## SpringSecurity 基本原理

SpringSecurity 本质是一个过滤器链：
从启动是可以获取到过滤器链：

```java
org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter
org.springframework.security.web.context.SecurityContextPersistenceFilter
org.springframework.security.web.header.HeaderWriterFilter
org.springframework.security.web.csrf.CsrfFilter
org.springframework.security.web.authentication.logout.LogoutFilter
org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter
org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter
org.springframework.security.web.savedrequest.RequestCacheAwareFilter
org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter
org.springframework.security.web.authentication.AnonymousAuthenticationFilter
org.springframework.security.web.session.SessionManagementFilter
org.springframework.security.web.access.ExceptionTranslationFilter
org.springframework.security.web.access.intercept.FilterSecurityInterceptor
```

FilterSecurityInterceptor：是一个方法级的权限过滤器, 基本位于过滤链的最底部。

![image-20220604142017641](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604142017641.png)

super.beforeInvocation(fi) 表示查看之前的filter 是否通过。
fi.getChain().doFilter(fi.getRequest(), fi.getResponse());表示真正的调用后台的服务。

ExceptionTranslationFilter：是个异常过滤器，用来处理在认证授权过程中抛出的异常

![image-20220604142029259](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604142029259.png)

UsernamePasswordAuthenticationFilter ：对/login的POST请求做拦截，校验表单中用户名，密码。

![image-20220604142037857](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604142037857.png)

**过滤器如何加载？**

核心类:**DelegatingFilterProxy**

![image-20220604143211715](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604143211715.png)

**两个重要接口**

1. **UserDetailsService接口讲解**

当什么也没有配置的时候，账号和密码是由Spring Security定义生成的。而在实际项目中账号和密码都是从数据库中查询出来的。 所以我们要通过自定义逻辑控制认证逻辑。
如果需要自定义逻辑时，只需要实现UserDetailsService接口即可。接口定义如下：

![image-20220604144018078](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604144018078.png)

* 返回值UserDetails

这个类是系统默认的用户“主体”

```java
// 表示获取登录用户所有权限 Collection<? extends GrantedAuthority> getAuthorities(); 
// 表示获取密码 String getPassword(); 
// 表示获取用户名 String getUsername(); 
// 表示判断账户是否过期 boolean isAccountNonExpired(); 
// 表示判断账户是否被锁定 boolean isAccountNonLocked(); 
// 表示凭证{密码}是否过期 boolean isCredentialsNonExpired(); 
// 表示当前用户是否可用 boolean isEnabled();
```

以下是UserDetails实现类

![image-20220604144105444](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604144105444.png)

以后我们只需要使用User 这个实体类即可！

![image-20220604144115533](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604144115533.png)

方法参数 username

表示用户名。此值是客户端表单传递过来的数据。默认情况下必须叫username，否则无法接收。

2. **PasswordEncoder 接口讲解**

```java
// 表示把参数按照特定的解析规则进行解析 String encode(CharSequence rawPassword); 
// 表示验证从存储中获取的编码密码与编码后提交的原始密码是否匹配。如果密码匹配，则返回true；如果不匹配，则返回false。第一个参数表示需要被解析的密码。第二个参数表示存储的密码。 boolean matches(CharSequence rawPassword, String encodedPassword); 
// 表示如果解析的密码能够再次进行解析且达到更安全的结果则返回true，否则返回false。默认返回false。 default boolean upgradeEncoding(String encodedPassword) { return false; }
```

接口实现类

![image-20220604144245364](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220604144245364.png)

BCryptPasswordEncoder是Spring Security官方推荐的密码解析器，平时多使用这个解析器。
BCryptPasswordEncoder是对bcrypt强散列方法的具体实现。是基于Hash算法实现的单向加密。可以通过strength控制加密强度，默认10.

查用方法演示

```java
@Test public void test01(){ 
    // 创建密码解析器 
    BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder(); 
    // 对密码进行加密 
    String atguigu = bCryptPasswordEncoder.encode("atguigu"); 
    // 打印加密之后的数据 
    System.out.println("加密之后数据：\t"+atguigu); 
    //判断原字符加密后和加密之前是否匹配 
    boolean result = bCryptPasswordEncoder.matches("atguigu", atguigu); 
    // 打印比较结果 
    System.out.println("比较结果：\t"+result); 
}
```





## 设置登录系统的账号、密码

方式一：在application.properties

```properties
spring.security.user.name=admin 
spring.security.user.password=admin
```

方式二：编写类实现接口

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {

        BCryptPasswordEncoder encoder=new BCryptPasswordEncoder();

        String password = encoder.encode("123");
        auth.inMemoryAuthentication().withUser("admin").password(password).roles("admin");
    }

}
```

**实现数据库认证来完成用户登录**

完成自定义登录

配置类:

```java
@Configuration
public class SecurityConfig02 extends WebSecurityConfigurerAdapter {

    @Autowired
    @Qualifier("userDetailsService")
    private UserDetailsService userDetailsService;

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }

}
```



```java
@Service("userDetailsService")
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private ITUserService userService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        QueryWrapper<TUser> wrapper = new QueryWrapper();
        wrapper.eq("username", username);
        TUser users = userService.getOne(wrapper);
        if (users == null) {
            throw new UsernameNotFoundException("用户名不存在！");
        }
        System.out.println(users);
        List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("role");
        return new User(users.getUsername(), new BCryptPasswordEncoder().encode(users.getPassword()), auths);
    }
}
```

## 自定义设置登陆页面

配置类中添加

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()  //自定义自己编写的页面
            .loginPage("login.html") //登陆页面设置
            .loginProcessingUrl("/user/login") //登录访问路径
            .defaultSuccessUrl("/test/index").permitAll()  //登陆成功之后 ，跳转路径
            .and().authorizeRequests()
                .antMatchers("/","/test/hello","/user/login").permitAll() //那些路径可以直接访问，不需要认证
            .anyRequest().authenticated()
            .and().csrf().disable(); //关闭csrf防护

    }
```

创建页面：

![image-20220606171559554](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606171559554.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <form action="/user/login" method="post">
        用户名: <input type="text" name="username">
        <br/>
        密码: <input type="password" name="password">
        <br/>
        <input type="submit" value="login">
    </form>
</body>
</html>
```

## 角色或权限进行访问控制

### hasAuthority方法

如果当前的主体具有指定的权限，则返回 true,否则返回false

当用户拥有指定的这一个权限才可以访问

![image-20220606172820764](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606172820764.png)

可以在登录校验 过滤器中给用户添加权限：
![image-20220606172633688](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606172633688.png)

### hasAnyAuthority方法

如果当前的主体有任何提供的角色（给定的作为一个逗号分隔的字符串列表）的话，返回true.

可以支持，拥有逗号分隔开来的其中一个权限，用户拥有就返回true

![image-20220606173108174](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606173108174.png)

### hasRole 方法

如果用户具备给定角色就允许访问,否则出现403。
如果当前主体具有指定的角色，则返回true。
底层源码：

![image-20220606173421288](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606173421288.png)

给用户添加角色：

![image-20220606173459798](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606173459798.png)

修改配置文件：
注意配置文件中不需要添加”ROLE_“，因为上述的底层代码会自动添加与之进行匹配。

![image-20220606173543490](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606173543490.png)

### hasAnyRole

和hasAnyAuthority方法类似。

## 自定义403页面

![image-20220606174126655](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606174126655.png)

## 注解使用

使用注解先要开启注解功能！

```java
@SpringBootApplication 
@EnableGlobalMethodSecurity(securedEnabled=true) 
public class DemosecurityApplication { 
    public static void main(String[] args) { SpringApplication.run(DemosecurityApplication.class, args); } 
}
```

### @Secured

判断是否具有角色，另外需要注意的是这里匹配的字符串需要添加前缀“ROLE_“。

用户具有某个角色，可以访问方法

```java
@GetMapping("/update")
    @Secured({"ROLE_sale","ROLE_manager"})
    public String update(){
        return "hello upfate!!";
    }
```

### @PreAuthorize

启动之前开启注解

```java
@SpringBootApplication
@MapperScan("com.zhy.secuirtydemo01.mapper")
@EnableGlobalMethodSecurity(securedEnabled = true,prePostEnabled = true)
public class Secuirtydemo01Application {

    public static void main(String[] args) {
        SpringApplication.run(Secuirtydemo01Application.class, args);
    }

}
```

```java
@PreAuthorize("hasAnyAuthority('admins')")
public String update(){
    return "hello upfate!!";
}
```

### @PostAuthorize

先开启注解功能：

```java
 @EnableGlobalMethodSecurity(prePostEnabled = true)
```

@PostAuthorize 注解使用并不多，在方法执行后再进行权限验证，适合验证带有返回值的权限.

```java
@GetMapping("/update")
@PostAuthorize("hasAnyAuthority('admins')")
public String update(){
    System.out.println("update...");
    return "hello upfate!!";
}
```

### @PostFilter


@PostFilter ：权限验证之后对数据进行过滤 留下用户名是admin1的数据

表达式中的 filterObject 引用的是方法返回值List中的某一个元素

```java
@RequestMapping("getAll")
@PreAuthorize("hasRole('ROLE_管理员')")
@PostFilter("filterObject.username == 'admin1'")
@ResponseBody
public List<UserInfo> getAllUser() {
    ArrayList<UserInfo> list = new ArrayList<>();
    list.add(new UserInfo(1l, "admin1", "6666"));
    list.add(new UserInfo(2l, "admin2", "888"));
    return list;
}
```

### @PreFilter

@PreFilter: 进入控制器之前对数据进行过滤

```java
@RequestMapping("getTestPreFilter")
@PreAuthorize("hasRole('ROLE_管理员')")
@PreFilter(value = "filterObject.id%2==0")
@ResponseBody
public List<UserInfo> getTestPreFilter(@RequestBody List<UserInfo> list) {
    list.forEach(t -> {
        System.out.println(t.getId() + "\t" + t.getUsername());
    });
    return list;
}
```

## 用户注销

### 在配置类中添加退出映射地址

```java
        http.logout().logoutUrl("/logout").logoutSuccessUrl("/test/hello").permitAll();
```

登陆退出按钮设置:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body> 登录成功<br> <a href="/logout">退出</a> </body>
</html>
```

## 基于数据库实现 记住我

原理图：

![image-20220606191044990](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606191044990.png)

### 创建表

```sql
CREATE TABLE `persistent_logins` (
`username` varchar(64) NOT NULL,
`series` varchar(64) NOT NULL,
`token` varchar(64) NOT NULL,
`last_used` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
PRIMARY KEY (`series`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

配置数据源

```java
 @Autowired
    private DataSource dataSource;

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        // 赋值数据源
         jdbcTokenRepository.setDataSource(dataSource);
        // 自动创建表,第一次执行会创建，以后要执行就要删除掉！
         jdbcTokenRepository.setCreateTableOnStartup(true);
         return jdbcTokenRepository;
    }
```

配置类中配置

![image-20220606192446205](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606192446205.png)

```java
 @Autowired
    private DataSource dataSource;

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        // 赋值数据源
         jdbcTokenRepository.setDataSource(dataSource);
        // 自动创建表,第一次执行会创建，以后要执行就要删除掉！
         jdbcTokenRepository.setCreateTableOnStartup(true);
         return jdbcTokenRepository;
    }
```

登录页面中添加复选框:

```html
记住我：<input type="checkbox"name="remember-me"title="记住密码"/><br/>
```



选中 复选框登陆成功后可以在 浏览器中看到cookie

![image-20220606193633776](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606193633776.png)

## CSRF 解释

​    跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。跟跨网站脚本（XSS）相比，XSS利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

 跨站请求攻击，简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。 从Spring Security 4.0开始，默认情况下会启用CSRF保护，以防止CSRF攻击应用程序，Spring Security CSRF会针对PATCH，POST，PUT和DELETE方法进行防护。

原理：就是认证后 服务端会生成一个csrftoken字符串保存在session信息中，每次请求过来，把csrftoken字符串信息作为参数发送给服务端，然后进行 与session中的csrftoken字符串进行比较

![image-20220606200727434](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220606200727434.png)

开启csrf防护 ，在页面中添加以下一句话

```html
<input type="hidden"th:if="${_csrf}!=null"th:value="${_csrf.token}"name="_csrf"/>
```

## SpringSecurity微服务权限方案

**SecurityContextPersistenceFilter过滤器**:前面提到过，在UsernamePasswordAuthenticationFilter过滤器认证成功之后，会在认证成功的处理方法中将已认证的用户信息对象 Authentication 封装进 SecurityContext，并存入 SecurityContextHolder。
之后，响应会通过SecurityContextPersistenceFilter过滤器，该过滤器的位置在所有过滤器的最前面，请求到来先进它，响应返回最后一个通过它，所以在该过滤器中处理已认证的用户信息对象 Authentication 与 Session 绑定。
认证成功的响应通过SecurityContextPersistenceFilter过滤器时，会从 SecurityContextHolder 中取出封装了已认证用户信息对象 Authentication 的 SecurityContext，放进 Session 中。当请求再次到来时，请求首先经过该过滤器，该过滤器会判断当前请求的 Session 是否存有 SecurityContext 对象，如果有则将该对象取出再次放入 SecurityContextHolder 中，之后该请求所在的线程获得认证用户信息，后续的资源访问不需要进行身份认证；当响应再次返回时，该过滤器同样从 SecurityContextHolder 取出 SecurityContext 对象，放入 Session 中。具体源码如下：

![image-20220608204502452](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220608204502452.png)

**SessionManagementFilter过滤器**: 在认证过滤器和授权过滤器的后面用于检查用户是否授权， 具体流程就是先检查session中有没有 springSecurityContextKey Key存在，有就放行。没有则检查 securityContext上下文有没有权限实例，有就放行.

权限管理数据模型

项目结构：



![image-20220607192448834](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220607192448834.png)



**项目源码地址:https://gitee.com/haoyumaster/guli_edu/tree/master/common/spring_security**
