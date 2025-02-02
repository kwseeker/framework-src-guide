# Quarkus 拓展机制

官方有很多资料：https://quarkus.io/guides/#q=extension

这里参考（从原理到实践）：

+ [Writing Your Own Extension](https://quarkus.io/guides/writing-extensions) （拓展的核心文档）
+ [CDI Integration Guide](https://quarkus.io/guides/cdi-integration)
+ [Build Items](https://quarkus.io/guides/all-builditems) （Quarkus 所有 BuildItem 类）
+ [A maturity matrix for Quarkus extensions](https://quarkus.io/guides/extension-maturity-matrix) （一个好的拓展的要求）
+ [Conditional Extension Dependencies](https://quarkus.io/guides/conditional-extension-dependencies) （条件拓展依赖）
+ [Quarkus Extension Metadata](https://quarkus.io/guides/extension-metadata) (Quarkus 拓展元数据， 介绍了包括 quarkus-extension.yaml 在内的五种文件的作用)
+ [Extension codestart](https://quarkus.io/guides/extension-codestart) （拓展代码生成系统，用于方便地构建一个扩展项目）
+ [Quarkus Extension Registry](https://quarkus.io/guides/extension-registry-user) （拓展注册表，记录社区贡献的拓展项目，可以给 Maven 等插件检索）
+ [Building my first extension](https://quarkus.io/guides/building-my-first-extension) （创建一个简单的 Servlet 拓展）
+ [Frequently asked questions about writing extensions](https://quarkus.io/guides/extension-faq)
+ [All configuration options](https://quarkus.io/guides/all-config) （Quarkus生态的一些配置选项）

> 其实看这些文档没多大卵用，仅仅了解个大概流程，怎么实现 BuildItemProcessor 文档根本说不清，比如 BuildItem 工作原理是什么从哪里获取的数据又传给了谁根本没有哪里有讲的，最后想彻底弄明白只能被逼着一点一点地啃源码。



## 简介

Quarkus 拓展包含两部分内容：**构建时增强**（buildtime augmentation）和**运行时容器**（runtime container），增强阶段的输出是记录的字节码，它负责直接实例化相关的运行时服务。

**Quarkus 应用程序包含三个引导阶段**：

+ 增强（Augmentation）

  由构建步骤处理器([Build Step Processors](https://quarkus.io/guides/writing-extensions#build-step-processors))完成。这些处理器可以访问 Jandex 注解信息，可以解析任何描述符并读取注解，但不应该尝试加载任何应用程序类。这些构建步骤的输出是一些记录的字节码，使用 ObjectWeb ASM 项目的扩展 Gizmo（ext/gizmo），在运行时用于实际引导应用程序。

+ 静态初始化（Static Init）

  如果字节码使用 `@Record(STATIC_INIT)` 记录，则它将在主类的静态初始化方法中执行。

+ 运行时初始化（Runtime Init）

  如果字节码使用 @Record(RUNTIME_INIT) 记录，则将从应用程序的主方法执行。

**Quarkus 拓展构建时的行为**：

1. 收集构建时间元数据并生成代码；
2. 执行基于应用程序紧密世界观的具有观点和合理的默认设置

3. 支持 Substrate VM 代码替换，以便库可以在 GraalVM 上运行
4. 主机宿主虚拟机代码替换，以帮助根据应用需求消除死代码
5. 发送元数据到 GraalVM，例如需要反射的示例类



## **创建自己的拓展**

需要结合一个实际的拓展看，比如 [redisson-quarkus](quarkus-extension-redisson.md)。

+ **构建一个拓展项目**

  可以使用 mvn 命令行构建。

  + **需要至少设置两个模块：deployment、runtime**

    这两个模块分别对应拓展的构建时增强和运行时容器。

    deployment: 部署时间子模块，用于处理构建时间处理和字节码记录， 需要依赖 `io.quarkus:quarkus-core-deployment`。

    runtime: 运行时子模块，包含将在本地可执行文件或运行时 JVM 中提供扩展行为的运行时行为，需要依赖 `io.quarkus:quarkus-core`。

  + **[Maven 配置](https://quarkus.io/guides/writing-extensions#using-maven)**

    需要包含 `io.quarkus:quarkus-extension-maven-plugin` 并配置 `maven-compiler-plugin` 以检测 `quarkus-extension-processor` 注解处理器，用于收集和生成必要的 Quarkus 扩展元数据，如果使用 Quarkus 父 pom，它将自动继承正确的配置。

    >  `maven-compiler-plugin` 配置需要 3.5+版本。

+ **BuildStepProcessor**

  **构建步骤**（Build Step）是一个带有 `@io.quarkus.deployment.annotations.BuildStep` 注解的非静态方法。

  **构建项**（Build Item）是抽象 `io.quarkus.builder.item.BuildItem` 类的具体、最终子类。每个构建项代表一些必须从一个阶段传递到另一个阶段的**信息单元**（比如构建时将数据库配置传递给下一阶段的组件）。

  调试可以看到每个构建步骤都在一个 `build-<x>` 线程中执行，`@Record(ExecutionTime.RUNTIME_INIT)`注释的构建步骤经过构建阶段后会被增强生成一个 StartupTask 类，在运行时执行。

  值生产。

  值消费。

  应用归档。

  使用线程上下文类加载器。

  使用 IndexDependencyBuildItem 将外部 JAR 添加到索引器。

  + **条件构建步骤**

    类似 Spring 条件注解。

+ **配置**

  通过 SmallRye Config 在 `application.properties` 中统一配置。扩展必须使用 SmallRye Config @ConfigMapping 来映射扩展所需的配置。这将允许 Quarkus 自动将映射实例暴露给每个配置阶段并生成配置文档。为了充分启用 Quarkus 最佳优化，最好将配置选项视为构建时确定而非运行时可覆盖。当然，像主机、端口、密码这样的属性应该在运行时可以覆盖。但许多属性，如启用缓存或设置 JDBC 驱动，可以安全地要求重新构建应用程序。

  可为配置设置**可用阶段**，共有三种阶段：BUILD_TIME、BUILD_AND_RUN_TIME_FIXED、RUN_TIME，参考 ConfigPhase.java。

  配置在不同的阶段注入到 Recorder 中（@Recorder注释的类）。

  如果扩展提供了额外的配置源，并且这些在静态初始化期间需要，则必须使用 **`StaticInitConfigBuilderBuildItem`** 进行注册。

  编写或生成配置文档。

+ **生成字节码**

  构建过程主要是会输出记录增强后的（@Recorder）的字节码。

  **RecoderContext**: 提供了一些方便的方法来增强字节码记录，这包括为没有无参构造函数的类注册创建函数的能力，注册对象替换（基本上是一个从不可序列化对象到可序列化对象以及相反的转换器），以及创建类代理。此接口可以直接作为方法参数注入到任何 `@Record` 方法中；@Record 注解用于标识一个类或方法应该被编译时处理并生成对应的字节码。

+ **上下文与依赖注入**

+ 拓展支持 Dev UI

+ 拓展定义端点

  扩展可以添加额外的非应用程序端点，以便与健康、指标、OpenAPI、Swagger UI 等端点一起提供服务，使用 `NonApplicationRootPathBuildItem` 定义。

+ **拓展支持开发模式**

  有各种 API 可供与开发模式集成，以及获取当前状态的信息，可以实现重启处理、实时重新加载。

+ **拓展的测试**

+ **通过 CDI 公开组件**

  扩展必须在构建时注册为用户应用程序易于消费的 bean，比如连接池框架暴露了 `DataSource` bean。

  扩展可以生成一个 **`AdditionalBeanBuildItem`** 来指示容器从类中读取 bean 定义，就像它是原始应用程序的一部分。

  一些 Bean 定义技巧：

  + @DefaultBean
  + SyntheticBeanBuildItem

