# Spring Security

此文档只是对流程图的补充。

Spring Security是一个框架，提供 [认证（authentication）](https://springdoc.cn/spring-security/features/authentication/index.html)、[授权（authorization）](https://springdoc.cn/spring-security/features/authorization/index.html) 和 [保护，以抵御常见的攻击](https://springdoc.cn/spring-security/features/exploits/index.html)。它对保护命令式和响应式应用程序有一流的支持，是保护基于Spring的应用程序的事实标准。

中文文档:

+ https://springdoc.cn/spring-security/

+ https://www.springcloud.cc/spring-security.html (单页)

  > 这个版本比较老，但是文档全部在一个网页，主要是方便查询。

官方案例: [spring-security-samples](https://github.com/spring-projects/spring-security-samples) (都是HelloWorld等级的项目，价值不大，还是推荐一些比较系统的开源项目)

参考开源项目：

+ [yudao-cloud](https://github.com/YunaiV/yudao-cloud)
+ [pre](https://github.com/LiHaodong888/pre)

> 不过个人感觉上面开源项目中整合Spring Security的代码中应该可以复用更多Spring Security 源码。



## Spring Security核心内容

这部分是看源码后，总结的个人感觉比较核心且常用的内容：

+ SpringSecurity 的本质

  一个拦截请求的**过滤器链**；分析的版本中提供了32个过滤器实现，主要提供三大功能：认证&授权&安全漏洞保护；

+ SpringSecurity 过滤器链装配到 SpringMVC（也可以是WebFlux）的方式和位置

  通过 **DelegatingFilterProxyRegistrationBean** 的方式注册并将 **springSecurityFilterChain** 装配在 **ApplicationFilterChain** 中；

+ 过滤器链创建原理

  参考 WebSecurity HttpSecurity 和 SecurityFilterChain 源码； 

+ 几个典型的功能（用户名密码认证、接口授权、OAuth、CAS等）

  + **认证**

    通过一个凭据（比如用户名密码、Token）换取用户信息（用户信息以及权限信息）用于后续权限校验。

    官方提供了一个用户名密码认证的例子：`UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter`；
    在认证成功后会生成 `Cookie`、`Session` 以及在安全上下文中保存 `UsernamePasswordAuthenticationToken`用于后续的授权校验；
    不过内部实现基于 **Cookie Session 机制**，现在生产环境多用 Token 机制，生产项目基本不会使用这个认证过滤器；

  + **授权**

    默认授权（权限校验）处理在 **FilterSecurityInterceptor** 完成；

  + ...

**生产项目中的方案设计**：

+ **登录校验&Token签发**

  常见的登录方式有用户名密码登录、手机号短信验证码登录、OAuth2第三方登录，登录验证通过后创建Token并返回给客户端；

  Token签发方式有自定义Token、JWT。

+ **Token认证**

  需要自行实现过滤器插入到 Spring Security 安全过滤器链中；

  包括Token有效性校验（是否存在、是否过期），校验Token有效后需要通过Token获取用户信息（JWT的话解密从中获取用户信息），将用户信息存储到安全上下文（通过SecurityContextHolder操作，用于后续资源访问的权限检查）。

+ **用户管理&权限管理**

  本身也属于资源访问部分，基于权限检查下对用户数据和权限数据的增删改查。

+ **OAuth2认证**



## Maven 依赖关系

Spring Boot 只需要引入 `spring-boot-starter-security`即可使用；

这个包，只是包含了一个 pom 文件，引入了下面依赖：

+ spring-boot-starter

  Spring Security 的自动配置类在这个包下定义。

  相关配置类：

  ```
  org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.oauth2.client.reactive.ReactiveOAuth2ClientAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.oauth2.resource.reactive.ReactiveOAuth2ResourceServerAutoConfiguration,\
  ```

+ spring-aop

+ spring-security-config

+ spring-security-web



## 配置类方法

> 同时也是功能特性的展示。

详细参考 `HttpSecurity`（注释非常详细且有示例代码） 和 `AuthenticationManagerBuilder` 源码。

### HttpSecurity

里面包含一组`HttpConfigurer`配置类，这些配置类其实就是用于初始化安全过滤器链中的安全过滤器的。

+ `headers()`支持在响应中添加安全相关的头

  + Cache-Control: no-cache, no-store, max-age=0, must-revalidate:
    no-cache: 表示浏览器不能使用缓存的副本，必须从原始服务器重新获取资源。
    no-store: 表示不允许缓存任何版本的页面。
    max-age=0: 表示资源在本地缓存中的有效期为0秒。
    must-revalidate: 表示浏览器必须验证缓存的状态，确保它仍然有效。

  + Pragma: no-cache:
    与Cache-Control: no-cache类似，指示浏览器不应使用缓存的版本，必须从服务器重新获取。

  + Expires: 0:
    指示浏览器不应使用缓存中的任何版本，因为资源已经过期。

  + X-Content-Type-Options: nosniff:
    防止浏览器执行一些特殊的 MIME 类型检测，确保浏览器按照服务器提供的Content-Type解释资源的类型，防止一些恶意的MIME类型绕过。

  + Strict-Transport-Security: max-age=31536000 ; includeSubDomains:
    启用严格的传输安全性（HSTS），告诉浏览器在指定的秒数内强制使用HTTPS连接。includeSubDomains指令表示HSTS也适用于所有子域。

  + X-Frame-Options: DENY:
    防止页面被嵌套到`<frame>`, `<iframe>`,` <embed>`, 或 `<object>` 中，提高防御点击劫持攻击的能力，

    安全策略除了`XFrameOptionsMode.DENY`还有`XFrameOptionsMode.SAMEORIGIN`。

  + X-XSS-Protection: 1; mode=block:
    启用浏览器内建的跨站脚本（XSS）过滤器。如果检测到XSS攻击，浏览器将阻止页面的加载。

+ `headers(Customizer<HeadersConfigurer<HttpSecurity>> headersCustomizer)`支持定制返回安全相关的头

  > 后面的方法都有对应的定制接口方法，原理是在Spring Security创建的`HttpConfigurer`对象上执行`Customizer`定制逻辑

+ `cors()`支持跨域

  这个和自定义的CORSFilter是重复的，可以这个就不需要自行实现了。

+ `sessionManagement()`支持会话管理，即存储Session，不过现在都是用Token机制

+ `portMapper()`支持HTTP和HTTPS请求重定向

+ `jee()`支持[预认证](https://springdoc.cn/spring-security/servlet/authentication/preauth.html)

+ `x509()`支持X.509证书认证，最常见的用途是在使用SSL时验证服务器的身份

+ `rememberMe()` 支持“记住我”功能

+ `authorizeRequests()`支持对URL进行访问限制，基于`HttpServletRequest`的`RequestMatcher`限制访问，最重要的一个方法

  ```java
  ExpressionInterceptUrlRegistry hasAnyAuthority(String... authorities)	//要求用户有列举的权限中任一一种权限
  ExpressionInterceptUrlRegistry hasAnyRole(String... roles)		//要求用户有列举的角色中任一一种角色
  ExpressionInterceptUrlRegistry hasAuthority(String authority)	//要求用户有某种权限
  ExpressionInterceptUrlRegistry hasIpAddress(String ipaddressExpression)	//要求请求来源IP位于IP白名单
  ExpressionInterceptUrlRegistry hasRole(String role)		//要求用户有某种角色
  ```

+ `authorizeHttpRequests()`支持对URL进行访问限制，基于`HttpServletRequest`的`RequestMatcher`限制访问

+ `requestCache()`支持请求缓存，比如用户还未认证情况下访问被保护页，会重定向到登录页，登录成功后再重定向到被保护页，请求缓存就是保存用户访问历史，开启`@EnableWebSecurity`注解会默认启用

+ `exceptionHandling()`支持异常处理（比如发生异常后重定向到访问被拒绝页面），开启`@EnableWebSecurity`注解会默认启用

+ `securityContext()`支持安全上下文的管理，开启`@EnableWebSecurity`注解会默认启用

+ `servletApi()`

+ `csrf()` 支持跨域请求伪造攻击防护，开启`@EnableWebSecurity`注解会默认启用，不过现在都用Token机制，不会有此风险，可以通过`csrf().disable()`关闭

  > Token机制不会像Cookie一样当请求某个网站时自动带上此网站的Cookie。 CSRF原理是第三方伪造网站，伪造网站内部访问可信任的网站，用户登录伪造网站，伪造网站请求可信任网站会自动将用户的Cookie带上去。

+ `logout()`支持注销登录，默认访问“/logout”，会使HttpSession无效，清理“rememberMe”认证信息、清理安全上下文（ SecurityContextHolder），最后跳转“/login?success”，开启`@EnableWebSecurity`注解会默认启用

+ `anonymous()`支持匿名访问，匿名访问用户默认会颁发`AnonymousAuthenticationToken`，包含角色`ROLE_ANONYMOUS`，开启`@EnableWebSecurity`注解会默认启用

+ `formLogin()`支持配置表单登录

+ `saml2Login() saml2Logout()`支持SAML2登录于退出登录

+ `oauth2Login()`支持 OAuth2登录

+ `oauth2Client() oauth2ResourceServer() oauth2ResourceServer()`支持OAuth2认证服务配置

+ `requiresChannel()`

+ `httpBasic()`支持Basic认证

+ `passwordManagement()`支持密码管理，比如修改密码

+ `authenticationManager(AuthenticationManager authenticationManager)` 修改默认的认证管理器

+ `authenticationProvider(AuthenticationProvider authenticationProvider)` 注册`AuthenticationProvider`

  用户认证信息提供者。

+ `userDetailsService(UserDetailsService userDetailsService)`注册`UserDetailsService`服务

  用户认证信息查询服务。

+ `addFilterAfter(Filter filter, Class<? extends Filter> afterFilter)`在某个过滤器（afterFilter）后面插入过滤器filter

+ `addFilterBefore(Filter filter, Class<? extends Filter> beforeFilter)`

+ `addFilter(Filter filter)`根据过滤器的order将过滤器插入filters合适的位置

+ `addFilterAt(Filter filter, Class<? extends Filter> atFilter) `插入同一个位置，原来的过滤器不会被覆盖

+ `requestMatchers()` 

+ `requestMatcher(RequestMatcher requestMatcher)`

+ `antMatcher(String antPattern)`

+ `mvcMatcher(String mvcPattern)`

+ `regexMatcher(String pattern)`

### AuthenticationManagerBuilder

### UserDetailsService



## HttpFirewall

作用：

+ 规范化请求URL, 便于匹配

  例如，一个原始的请求路径为 `/secure;hack=1/somefile.html;hack=2`，被返回为 `/secure/somefile.html`

+ 防止 HTTP 响应拆分（HTTP Response Splitting，HRS）攻击

  也称为CRLF注入漏洞，恶意攻击者将CRLF换行符加入到请求中，从而使一个请求产生两个响应，前一个响应是服务器的响应，而后一个则是攻击者设计的响应。

  参考：[CRLF注入（HTTP响应拆分/截断）](https://zhuanlan.zhihu.com/p/617476453)

+ 防止 [跨站追踪（XST）](https://www.owasp.org/index.php/Cross_Site_Tracing) 和 [HTTP Verb Tampering](https://www.owasp.org/index.php/Test_HTTP_Methods_(OTG-CONFIG-006))



## [异步请求处理](https://springdoc.cn/spring-security/servlet/integrations/mvc.html#mvc-async)

Spring Web MVC 3.2+对 异步请求处理有很好的支持。不需要额外的配置，Spring Security会自动将 SecurityContext 设置为调用 Controller 返回的 Callable 的 Thread。例如，下面的方法会自动用创建 Callable 时可用的 SecurityContext 来调用它的 Callable。

```java
@RequestMapping(method=RequestMethod.POST)
public Callable<String> processUpload(final MultipartFile file) {
    return new Callable<String>() {
        public Object call() throws Exception {
        // ...
        return "someView";
        }
    };
}
```

Spring MVC 异步请求场景包括：

1. 处理需要长时间才能完成的请求。例如：搜索引擎模糊搜索、大文件下载、视频文件转换等。
2. 处理大量并发请求。例如：高并发的在线聊天、商品秒杀活动等。

参考：[高性能关键技术之---体验Spring MVC的异步模式（Callable、WebAsyncTask、DeferredResult） 基础使用篇](https://cloud.tencent.com/developer/article/1497796)

