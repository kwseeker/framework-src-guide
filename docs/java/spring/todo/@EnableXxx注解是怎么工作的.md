# @EnableXxx 注解工作原理

Spring全家桶源码中有大量的`@EnableXxx`注解，这种注解一般都包含`@Import`注解用于加载其他配置类、以及`ImportSelector`；

但是`@EnableXxx`注解本身又是怎么被扫描并被处理的（注意不是讨论它引入的`@Import`是怎么被处理的）？这个疑问已经存在很久了，这里记录下。

比如`Spring Security OAuth2`的`@EnableAuthorizationServer`：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AuthorizationServerEndpointsConfiguration.class, AuthorizationServerSecurityConfiguration.class})
public @interface EnableAuthorizationServer {
}
```

之前看`@Configuration`、`@Import`注解的处理逻辑的源码，会有类专门扫描加载的类上是否有这些注解；

但是在源码中并没有搜索到哪里有扫描`@EnableXxx` 注解，比如上面的例子我在`Spring Security OAuth2`的源码中没有找到哪里有扫描`EnableAuthorizationServer.class`; 

搜索资料，基本都是直接忽视`@EnableXxx` 然后将 `@EnableXxx` 引入的 `@Import` 是怎么工作的。

没什么时间去回顾源码，但是我做了一个假设：**猜测扫描配置类的处理器（比如`ConfigurationClassPostProcessor`）可能会递归扫描配置类上所有注解内部引入的注解是否包含`@Import` 、`@ImportResource`等注解**；

做了一个简单的测试：

```java
// 自定义@EnableXxx注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(MyConfiguration.class)
public @interface EnableForTest {
}

//自定义配置类
public class MyConfiguration {
    @Bean
    public String myName() {
        System.out.println("calling MyConfiguration#myName()");
        return "Arvin";
    }
}

//在配置类上添加自定义@EnableXxx注解（加在任意配置类上均可，不需要是主类）
@EnableForTest
@SpringBootApplication
public class AuthorizationServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(AuthorizationServerApplication.class, args);
    }
}
```

经测试是可以加载自定义配置类的，说明**自定义`@EnableXxx`注解的名字不重要，只要里面有`@Import`就能将`@Import`引入的配置类扫描到**。

后面有空看下源码验证下猜想（TODO）。