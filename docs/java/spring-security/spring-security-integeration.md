# Spring Boot 集成 Spring Security

长时间不看的内容总是容易遗忘，这一篇文章为了每次集成这个框架都能快速回忆起内部原理，快速完成集成。

集成框架到系统和学习框架关注点不同，学习框架更多关注**整体结构、工作原理、实现细节**；集成框架时更多关注**根据业务需求判断自己应该对框架里面哪些部分作出改动**。

> 尽量自己多参考开源项目、官方文档将各个功能点都自己尝试集成一边，并记录到下面的"功能备忘"中，同时测试DEMO也可以作为日后集成时的参考。



**集成流程**：

1. 首先看流程图快速回忆起Spring Security内部工作原理；
2. 结合下面的“功能备忘”定制功能，各个功能可以先写个壳在后面边调试边完善；
3. 测试登录请求发放Token；
4. 带有权限控制的请求传参Token并发送；
5. Token 认证过滤器，Token的解析，获取用户信息（解析流程可能涉及调用用户中心的接口）
6. Token解析后的用户信息重新封装成`AbstractAuthenticationToken`并保存到Spring Security 安全上下文 `SecurityContextHolder`；
7. `FilterSecurityInterceptor` 过滤器中执行自定义的权限校验（比如通过 @PreAuthorize()调用Bean方法检查权限）；



## 可能需要定制的功能备忘

+ **配置类中创建过滤器链（SecurityFilterChain） **

  Spring Security 的核心就是一组过滤器链，集成的第一步就是配置这个过滤器链；

  通过 HttpSecurity Bean 进行配置， HttpSecurity 其实是一组过滤器的Builder, Spring Security 自动配置阶段会自动创建 HttpSecurity Bean, 里面还包含了一些默认注册的过滤器链。

  ```java
  @Bean
  protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
      ...
      return httpSecurity.build();
  }
  ```

  常用的过滤器配置：

  + cors()

  + csrf()

    如果不使用 Session, 可以关闭这个配置，就不会注册防止CSRF的过滤器了；

    ```
    .csrf().disable()
    ```

  + sessionManagement()

    现在一般都采用Token机制，不使用Session了，可以关闭Session管理；

    ```
    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
    ```

  + headers()

  + exceptionHandling() 异常处理

    经过过滤器链处理，可能抛出异常（`AccessDeniedException`、`SpringSecurityException`），异常会被 ExceptionTranslationFilter 捕获并处理，exceptionHandling() 就是用于定义 ExceptionTranslationFilter 的异常处理逻辑。

    ```java
    .exceptionHandling()
        .authenticationEntryPoint(authenticationEntryPoint()) 	// SpringSecurityException异常处理器，此异常表示未通过身份验证，因此服务器会发回一个响应，指示必须进行身份验证，或重定向到特定的web页面
        .accessDeniedHandler(accessDeniedHandler()).and()		// AccessDeniedException 异常处理器，通常是返回一个错误响应，表示访问被拒绝
    ```

  + authorizeRequests()

    配置请求认证规则，比如配置不需要认证的请求类型和路径；

    AuthorizeRequestsCustomizer 可以用于配置子模块的特殊认证规则。

    ```java
    .authorizeRequests()	//配置不需要认证的请求路径
    	.antMatchers(HttpMethod.GET, "/*.html", "/**/*.html", "/**/*.css", "/**/*.js").permitAll()
        .and()
    .authorizeRequests(specialAuthorizeRequestsCustomizer()) //通过 AuthorizeRequestsCustomizer 配置的规则
    .authorizeRequests()
        .anyRequest().authenticated()	//除了上面配置的规则，其他都需要认证
    ```

+ **自定义认证过滤器**

  官方提供的认证过滤器有用户名密码过滤器，但是现实业务中首次登录使用用户名密码之后都使用Token认证；

  所以需要自定义 Token认证的过滤，一般放在 UsernamePasswordAuthenticationFilter 前面，另外如果使用官方的 UsernamePasswordAuthenticationFilter 实现用户名密码登录的话，需要对这个过滤器的组件进行修改，也可以自行实现登录逻辑不使用 UsernamePasswordAuthenticationFilter。

  ```
  httpSecurity.addFilterBefore(tokenAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
  ```

+ **方法权限校验配置**

  更多关于方法安全控制：https://www.springcloud.cc/spring-security.html#jc-method

  + **在配置类上添加注解 @EnableGlobalMethodSecurity  开启方法的权限访问控制**

      这个注解可以开启 `@PreAuthorize() `、`@Secured()` 等注解的权限访问控制；
      
      `@PreAuthorize()`支持[基于表达式访问控制](https://www.springcloud.cc/spring-security.html#el-access)，支持基于角色、权限、用户主体对象的权限判断。
      
      ```java
      @EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
      ```

  + **自定义权限检查**

    由于业务代码可能有自行配置 RBAC 等权限控制方案，需要对接到 Spring Security  @PreAuthorize; 

    所以需要自定义权限检查，由于 @PreAuthorize() 支持 SpEL，而 SpEL 中是可以包含方法调用的；

    所以只需要自定义一个方法权限检查类（系统启动时创建Bean）即可，请求时通过 @PreAuthorize 调用Bean的方法检查权限。

+ **密码编码器**

  为了防止用户密码泄漏，在传输和数据库存储中一般都使用加密后的密码；

  ```java
  @Bean
  public PasswordEncoder passwordEncoder() {
  	return new BCryptPasswordEncoder(msaSecurityProperties.getPasswordEncoderLength());
  }
  ```



## 集成时遇到的坑

+ **`FilterChainProxy`中有两条过滤器链**

  版本：spring-boot-starter-security:2.7.12 - 2.7.14 都碰到过，其他版本暂没测试。

  其中一条是框架自动配置类中默认创建的（表单登录、BASIC认证），另一个是自己手动配置创建的；

  由于默认的过滤器链配置的所有请求均需要经过验证，所以会匹配我们系统中的所有请求路径，即所有请求都会被这条默认创建的过滤器所拦截，但是这个默认的过滤器链里面配置的流程肯定走不通最终报错。

  默认创建的过滤器链的源代码：

  ```java
  @Configuration(
      proxyBeanMethods = false
  )
  @ConditionalOnWebApplication(
      type = Type.SERVLET
  )
  class SpringBootWebSecurityConfiguration {
  	//... 
  	
      //这个条件注解主要是说Bean WebSecurityConfigurerAdapter、SecurityFilterChain 不存在就创建
  	@ConditionalOnDefaultWebSecurity
      static class SecurityFilterChainConfiguration {
          SecurityFilterChainConfiguration() {
          }
  
          @Bean
          @Order(2147483642)
          SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
              //这里可以看到配置的是所有请求都必须经过认证
              ((ExpressionUrlAuthorizationConfigurer.AuthorizedUrl)http.authorizeRequests()
  .anyRequest()).authenticated();
              http.formLogin();
              http.httpBasic();
              return (SecurityFilterChain)http.build();
          }
      }
  }
  ```

  自己手动配置的过滤器链：

  ```java
  @Bean
  //@Order(2147483641) //想要在 defaultSecurityFilterChain 之前创建但是测试发现无效
  protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
  	httpSecurity.
          ...
      return httpSecurity.build();
  }
  ```

  **最终解决**：

  在自己的配置类上添加 `@AutoConfigureBefore(SecurityAutoConfiguration.class)`。

  > 关于 @Order 为何无效，测试了加载配置类上也无效，没有找到好的资料作出完美解释，有空了自己研究源码吧，Spring 主流程的源码已经看了一遍，只看这个问题应该不复杂。
  >
  > 不过还是贴下别人的分析：[踩坑！@Order失效。。。](https://blog.csdn.net/qq_34142184/article/details/126951618?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-126951618-blog-125897498.235%5Ev43%5Epc_blog_bottom_relevance_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-126951618-blog-125897498.235%5Ev43%5Epc_blog_bottom_relevance_base2&utm_relevant_index=5)

  
