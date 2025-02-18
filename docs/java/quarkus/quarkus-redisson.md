# 从 redisson-quarkus 源码看集成 Redisson 时如何修改编解码器

redisson-quarkus 属于 redisson 中的一个模块，用于在 Quarkus 中集成 Redisson 。

**runtime** 中定义Quarkus拓展业务实现，**deployment** 生成注册字节码，**integration-tests** 模拟应用程序集成当前拓展进行测试（可以当作 Demo)。

其实 redisson-quarkus  integration-tests 中已经展示了怎么修改编解码器，不过还是应该研究下怎么配置是怎么生效的；从后面的代码以及文档分析看，应该是先将配置加载到了拓展组件定义的配置类 RedissonConfig 然后统一加入到了 SmallRyeConfig 中（待验证），之后组件通过 ConfigProvider 以及配置项前缀过滤读取自己需要的配置。

```properties
quarkus.redisson.codec=org.redisson.codec.Kryo5Codec
# 改成 JsonJacksonCodec
quarkus.redisson.codec=org.redisson.codec.JsonJacksonCodec
```

redisson-quarkus 项目结构：

```shell
cdi
├── deployment
│   ├── pom.xml
│   └── src
│       └── main
│           └── java
│               └── io.quarkus.redisson.client.deployment
│                                              ├── QuarkusRedissonClientProcessor.java # 构建步骤处理器，执行多个构建步骤创建多个构建项
│                                              └── RedissonClientItemBuild.java
├── integration-tests
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── org.redisson.quarkus.client.it
│       │   │                                   ├── QuarkusRedissonClientResource.java
│       │   │                                   ├── RemoteServiceImpl.java
│       │   │                                   ├── RemService.java
│       │   │                                   └── Task.java
│       │   └── resources
│       │       ├── application.properties
│       │       ├── config.xml
│       │       └── reflection-config.json
│       └── test
│           └── java
│               └── org.redisson.quarkus.client.it
│                                               ├── NativeQuarkusRedissonClientResourceIT.java
│                                               └── QuarkusRedissonClientResourceTest.java
├── pom.xml
└── runtime
    ├── pom.xml
    └── src
        └── main
            ├── java
            │   └── io.quarkus.redisson.client.runtime
            │                                  ├── graal
            │                                  │   ├── ByteBuddySubstitutions.java
            │                                  │   └── CodecsSubstitutions.java
            │                                  ├── RedissonClientProducer.java  # 用于生成 RedissonClient 实例的 Producer，定义 RedissonClient 创建流程，被定义为了ApplicationScoped， 可以将其当作 Spring 的 AutoConfiguration 类
            │                                  ├── RedissonClientRecorder.java	# 被 @Recorder 注解标识，表示这是一个构建时记录器，在 Deployment 阶段被增强
            │                                  └── RedissonConfig.java
            └── resources
                └── META-INF
                    ├── native-image
                    │   └── org.redisson
                    │       └── redisson
                    │           └── serialization-config.json
                    └── quarkus-extension.yaml # 这个文件必须包含；当使用 Quarkus 开发工具构建、测试或编辑 Quarkus 应用程序项目时，Quarkus 扩展 JAR 软件包将通过其中存在的 Quarkus 扩展元数据文件在应用程序类路径中被识别，Quarkus 将在构建时生成；详细参考：https://quarkus.io/guides/extension-metadata#quarkus-extension-yaml
```



## 代码分析

### deployment

+ QuarkusRedissonClientProcessor

  ```java
  @BuildStep
  AdditionalBeanBuildItem addProducer() {
      // 用于指定在 bean 发现期间要分析的一个或多个额外的 bean 类；默认情况下，如果认为这些 bean 未使用并且启用了 ArcConfig#removeUnusedBeans ，则生成的 bean 可能会被移除。您可以通过设置 #removable 到 false 以及通过 Builder#setUnremovable() 来更改默认行为。这里指定 RedissonClientProducer 不可被移除
      return AdditionalBeanBuildItem.unremovableOf(RedissonClientProducer.class);
  }
  
  @BuildStep
  void addConfig(BuildProducer<NativeImageResourceBuildItem> nativeResources,
                 BuildProducer<HotDeploymentWatchedFileBuildItem> watchedFiles,
                 BuildProducer<RuntimeInitializedClassBuildItem> staticItems,
                 BuildProducer<ReflectiveClassBuildItem> reflectiveItems) {
      nativeResources.produce(new NativeImageResourceBuildItem("redisson.yaml"));
      nativeResources.produce(new NativeImageResourceBuildItem("META-INF/services/org.jboss.marshalling.ProviderDescriptor"));
      // 记录热部署文件，如果修改，在开发模式下会导致热重新部署
      watchedFiles.produce(new HotDeploymentWatchedFileBuildItem("redisson.yaml"));
      // ReflectiveClassBuildItem 用于在原生模式（native mode）下注册类以供反射使用
      reflectiveItems.produce(ReflectiveClassBuildItem.builder(Kryo5Codec.class)
                              .methods(false)
                              .fields(false)
                              .build()
                             );
      reflectiveItems.produce(ReflectiveClassBuildItem.builder(
          RemoteExecutorService.class,
          RemoteExecutorServiceAsync.class)
                              .methods(true)
                              .fields(false)
                              .build()
                             );
      reflectiveItems.produce(ReflectiveClassBuildItem.builder(
          Config.class,
          BaseConfig.class,
          BaseMasterSlaveServersConfig.class,
          SingleServerConfig.class,
          ReplicatedServersConfig.class,
          SentinelServersConfig.class,
          ClusterServersConfig.class)
                              .methods(true)
                              .fields(true)
                              .build()
                             );
      reflectiveItems.produce(ReflectiveClassBuildItem.builder(
          RBucket.class,
          RedissonBucket.class,
          RedissonObject.class,
          RedissonMultimap.class)
                              .methods(true)
                              .fields(true)
                              .build()
                             );
      reflectiveItems.produce(ReflectiveClassBuildItem.builder(
          RObjectReactive.class,
          RExpirable.class,
          RObject.class)
                              .methods(true)
                              .build()
                             );
  }
  
  // 这个方法分两部处理：
  // 1 构建时增强生成一个 StartupTask 类，如 QuarkusExtensionRedissonProcessor$build620656674.class，这个类 deploy_0() 方法内执行 recorder.createProducer()
  // 2 在运行时执行 StartupTask 类的 deploy_0() 方法，执行 recorder.createProducer(), 创建 RedissonClientProducer Bean
  // RedissonClientItemBuild 继承 SimpleBuildItem
  @BuildStep
  @Record(ExecutionTime.RUNTIME_INIT)
  RedissonClientItemBuild build(RedissonClientRecorder recorder) throws IOException {
      recorder.createProducer();
      return new RedissonClientItemBuild();
  }
  // QuarkusExtensionRedissonProcessor$build620656674.class
  //    public void deploy(StartupContext var1) {
  //        var1.setCurrentBuildStepName("QuarkusExtensionRedissonProcessor.build");
  //        Object[] var2 = this.$quarkus$createArray();
  //        this.deploy_0(var1, var2);
  //    }
  //
  //    public void deploy_0(StartupContext var1, Object[] var2) {
  //        (new RedissonClientRecorder()).createProducer();
  //    }
  ```
  

### runtime

+ pom.xml

  内部添加了 org.graalvm.nativeimage:svm 依赖用于支持原生镜像编译。

+ META-INF/quarkus-extension.yaml

  参考： https://quarkus.io/guides/extension-metadata#quarkus-extension-yaml

  里面对开发最重要的信息有：

  ```yaml
  metadata:
  	config:	# 配置前缀
  		- "quarkus.redisson"
  ```

+ RedissonConfig.java

  定义了几组运行时配置：

  ```java
  @ConfigRoot(phase = ConfigPhase.RUN_TIME) //可以将应用程序的配置项集中在一个类中，并且这些配置项可以自动从外部配置源（如环境变量、配置文件等）加载
  @ConfigMapping(prefix = "quarkus")
  public interface RedissonConfig {
  	// 这里可以看到是直接将所有以 quarkus.redissson 开头的配置全部注入到了 Map
      @WithName("redisson")
      Map<String, String> params();
  
      @WithName("redisson.single-server-config")
      Map<String, String> singleServerConfig();
  
      @WithName("redisson.cluster-servers-config")
      Map<String, String> clusterServersConfig();
  
      @WithName("redisson.sentinel-servers-config")
      Map<String, String> sentinelServersConfig();
  
      @WithName("redisson.replicated-servers-config")
      Map<String, String> replicatedServersConfig();
  
      @WithName("redisson.master-slave-servers-config")
      Map<String, String> masterSlaveServersConfig();
  }
  ```


+ RedissonClientRecorder.java

  ```java
  @Recorder
  public class RedissonClientRecorder {
  
      public void createProducer() {
          Arc.container().instance(RedissonClientProducer.class).get();
      }
  }
  ```

+ RedissonClientProducer

  这个类可以当作 Spring 的 AutoConfiguration 类。

  ```java
  @ApplicationScoped
  public class RedissonClientProducer {
  
      private RedissonClient redisson;
  
      @Inject
      public ShutdownConfig shutdownConfig;
  
      @Produces	// 类似Spring @Bean 注解
      @Singleton
      @DefaultBean
      public RedissonClient create() throws IOException {
      	...
      }
      
      ...
  }
  ```
