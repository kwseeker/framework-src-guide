# [Dubbo SPI](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/spi/overview/)

版本：3.2.9，源码约3K行，提供了丰富的单元测试。

Dubbo SPI的数据结构和工作原理主流程图。

![dubbo3-spi](imgs/dubbo3-spi.png)

官方文档对SPI部分的描述，感觉不够详细，这里补充从源码总结出来的没有详细说明的知识点。



## 常用知识点

+ 注解

  + **@SPI**

    标注在拓展类接口上，用于表示需要通过Dubbo SPI机制加载接口实现类。

    实现类通过SPI文件列举，位于 `META-INF/dubbo/`。

    ```java
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE})
    public @interface SPI {
    	//默认的拓展实现类名，对应SPI文件中的key
        String value() default "";
    	//共享范围，如果有父ExtensionDirector默认由scope相同的父ExtensionDirector加载，
        ExtensionScope scope() default ExtensionScope.APPLICATION;
    }
    ```

  + **@Adaptive**

    注解在类上，用于表示这个类是手动实现的自适应的拓展类；

    自适应拓展类其实类似代理类，真正干活的还是其他实现类；

    注解在接口方法上，用于表示由Dubbo自动生成自适应拓展类。

    **通过这个注解其实是定义一个拓展类选择器，根据规则选择一个拓展实现类进行处理。**

    ```java
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    public @interface Adaptive {
    	//例如，给定 String[] {"key1", "key2"}：
    	//在 URL 中找到参数“key1”，将其值用作扩展名
    	//如果在 URL 中找不到“key1”（或其值为空），请尝试“key2”作为扩展名
    	//如果“key2”也不存在，则使用默认扩展名
    	//否则，抛出 IllegalStateException
        String[] value() default {};
    }
    ```

    > 官方单元测试：ExtensionLoader_Adaptive_Test.java
    >
    > ```java
    > 
    > @SPI("impl1")
    > public interface UseProtocolKeyExt {
    >  //这个注解结合@SPI("impl1")
    >  //最终的效果是先按url参数key1的值选择实现类，key1的值对应的实现类不存在就按url协议选择，协议也存在就使用默认拓展名impl1选择实现类
    >  @Adaptive({"key1", "protocol"})
    >  String echo(URL url, String s);
    > 	// ...
    > }
    > 
    > //对应生成的自适应拓展类源码，可以看到和@Adaptive定义的规则一致
    > public class UseProtocolKeyExt$Adaptive implements UseProtocolKeyExt {
    > 
    >  public java.lang.String echo(URL arg0, java.lang.String arg1) {
    >      if (arg0 == null)
    >          throw new IllegalArgumentException("url == null");
    >      URL url = arg0;
    >      //选择拓展类名规则：key1 -> protocol -> impl1
    >      String extName = url.getParameter("key1", (url.getProtocol() == null ? "impl1" : url.getProtocol()));
    >      if (extName == null)
    >          throw new IllegalStateException("Failed to get extension (org.apache.dubbo.common.extension.ext3.UseProtocolKeyExt) name from url (" + url.toString() + ") use keys([key1, protocol])");
    >      ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), UseProtocolKeyExt.class);
    >      //通过拓展类名获取实例
    >      UseProtocolKeyExt extension = (UseProtocolKeyExt) scopeModel.getExtensionLoader(UseProtocolKeyExt.class)
    >          .getExtension(extName);
    >      return extension.echo(arg0, arg1);
    >  }
    > 
    >  //...
    > }
    > ```

    AdaptiveClassCodeGenerator 代码生成原理，这里只关注方法生成原理：

    看单元测试代码发现所有自适应的接口方法第一个参数都是`org.apache.dubbo.common.URL`，一度以为所有自适应接口方法第一个参数都必须是这个类型，但是后面发现并不是，那如果自己拓展接口，参数应该怎么定义？还是需要看源码研究一下参数定义规则。

    主要看 `AdaptiveClassCodeGenerator#generateMethodContent(Method method)`的实现原理。

    结论：**自适应方法参数列表中要么包含URL参数，要么有参数包含能返回URL的方法**。
  
    比如 `ProxyFactory`自适应方法 `<T> T getProxy(Invoker<T> invoker) throws RpcException;`的参数类型 `Invoker `内部包含 `URL getUrl()`方法可以放回URL对象。
  
    ```java
    @SPI(value = "javassist", scope = FRAMEWORK)
    public interface ProxyFactory {
        @Adaptive({PROXY_KEY})
        <T> T getProxy(Invoker<T> invoker) throws RpcException;
        //...
    }
    ```
  
  + **@Activate**
  
    按条件激活拓展类。有点类似 Spring 条件注解。
  
    主要用在`Filter`上，比如有的`Filter`需要在`provider`执行，有的需要在`consumer`执行，根据`url`中的参数指定和`group`(`provider`还是`consumer`)，运行时决定哪些`filter`需要被引入执行。
  
    ```java
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    public @interface Activate {
        /**
         * 有一个group匹配 ExtensionLoader#getActivateExtension(URL url, String key, String group) 传参的group时激活
         */
        String[] group() default {};
    
        /**
         * 比如@Activate("cache, validation")，还支持 @Activate("cache:redis, validation")这种写法，存储方式略有不同
         * ExtensionLoader#getActivateExtension 传参key对应URL中的value出现在 value数组中激活
         * @return URL parameter keys
         * @see ExtensionLoader#getActivateExtension(URL, String)
         * @see ExtensionLoader#getActivateExtension(URL, String, String)
         */
        String[] value() default {};
    
        @Deprecated
        String[] before() default {};
        @Deprecated
        String[] after() default {};
    
        /**
         * 用于ExtensionLoader#getActivateExtension返回值列表排序
         */
        int order() default 0;
    
        /**
         * 数组中的类存在则激活
         */
        String[] onClass() default {};
    }
    ```
  
    > 官方单元测试：ExtensionLoader_Activate_Test.java
    >
    > ```java
    > @SPI("impl1")
    > public interface ActivateExt1 {
    >     String echo(String msg);
    > }
    > 
    > @Test
    > void testActiveByValue() {
    >     ExtensionLoader<ActivateExt1> loader = ExtensionLoader.getExtensionLoader(ActivateExt1.class);
    >     URL url = URL.valueOf("test://localhost/test?keyA=value");
    >     //url中参数keyA的值是value
    >     //即查询group()包含value且value()包含value的拓展类的实例
    >     List<ActivateExt1> list2 = loader.getActivateExtension(url, "keyA", "value");
    > }
    > ```
  
  + **@Wrapper**
  
    用于定义AOP包装类。
  
  + **@DisableInject**
  
    禁止对拓展类实例的字段进行IOC属性依赖注入。



