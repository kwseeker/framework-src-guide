# [JCasbin](https://casbin.org/zh/docs/overview)

学习资料：[Tutorials](https://casbin.org/zh/docs/tutorials)

Casbin [editor](https://casbin.org/editor/)

Casbin是一个强大且高效的开源访问控制库，支持各种[访问控制模型](https://en.wikipedia.org/wiki/Access_control#Access_control_models)，用于在全局范围内执行授权。

JCasbin 是其 Java 语言实现。

**注意 Casbin 只做访问控制（权限校验），不做身份验证和管理用户、角色列表**。

关于身份验证和管理用户、角色，其实 Spring Security 也基本只提供了身份认证的接口，并没有提供企业级的身份认证和用户角色管理的实现。

入门开源DEMO项目：

+ [casbin-spring-boot-example](https://github.com/jcasbin/casbin-spring-boot-example) (官方的示例)

  整合到了Spring Security，但是里面做了重复的权限校验。

+ [jcasbin-springboot-demo](https://github.com/VINO42/jcasbin-springboot-demo)

  这个项目主要是展示使用MySQL存储模型和策略数据。



## 支持的访问控制模型

+ [**ACL (访问控制列表)**](https://en.wikipedia.org/wiki/Access_control_list)
  + **带有[超级用户](https://en.wikipedia.org/wiki/Superuser)的ACL**
  + **无用户的ACL**：这对于没有身份验证或用户登录的系统特别有用。
  + **无资源的ACL**：在某些情况下，目标是一种资源类型，而不是单个资源。 可以使用像"write-article"和"read-log"这样的权限。 这并不控制对特定文章或日志的访问。
+ **[RBAC (基于角色的访问控制)](https://en.wikipedia.org/wiki/Role-based_access_control)**
  + **带有资源角色的RBAC**：用户和资源同时可以拥有角色（或组）。
  + **带有域/租户的RBAC**：用户可以为不同的域/租户拥有不同的角色集。
+ **[ABAC (基于属性的访问控制)](https://en.wikipedia.org/wiki/Attribute-Based_Access_Control)**：可以使用类似"resource.Owner"的语法糖来获取资源的属性。

 

## 基本概念

+ **PERM元模型（策略，效果，请求，匹配器）**

  + **请求 r**

    如：`r={sub,obj,act}`。

    + **主体 sub**

      访问实体，请求发出人或请求发出人的角色。

    + **对象 obj**

      被访问资源。

    + **动作 act**

      访问方法。

  + **策略 p**

    指定了策略规则文档中字段的名称和顺序，如：`p={sub, obj, act}` 或 `p={sub, obj, act, eft}`。

    + **效果 eft**

      allow（默认值）、deny。

  + **角色 g**

    用于定义 RBAC 角色继承关系。

  + **策略效果 e**

    对匹配器的匹配结果进行逻辑组合判断。

    内置了5种策略效果：

    | 策略效果                                                     | 含义             | 例子                                                         |
    | ------------------------------------------------------------ | ---------------- | ------------------------------------------------------------ |
    | some(where (p.eft == allow))                                 | allow-override   | [ACL, RBAC, etc.](https://casbin.org/zh/docs/supported-models#examples) |
    | !some(where (p.eft == deny))                                 | deny-override    | [Deny-override](https://casbin.org/zh/docs/supported-models#examples) |
    | some(where (p.eft == allow)) && !some(where (p.eft == deny)) | allow-and-deny   | [Allow-and-deny](https://casbin.org/zh/docs/supported-models#examples) |
    | priority(p.eft) \|\| deny                                    | priority         | [Priority](https://casbin.org/zh/docs/supported-models#examples) |
    | subjectPriority(p.eft)                                       | 基于角色的优先级 | [主题-优先级](https://casbin.org/zh/docs/supported-models#examples) |

  + **匹配器 m**

    用于匹配请求和策略。

    匹配器中可以使用算术运算符如`+, -, *, /`和逻辑运算符如`&&, ||, !`；

    还内置了一些匹配函数，参考：[匹配器中的函数](https://casbin.org/zh/docs/function)，另外也支持自定义匹配函数。

+ **[模型语法](https://casbin.org/zh/docs/syntax-for-models)**

  上面模型的详细定义。

+ **注意事项**

  如果模型中定义多组request_definition、policy_definition、policy_effect、matchers， 在某次权限校验时多组定义并不会被同时使用，可以通过 EnforceContext() 指定使用哪一组。

  ```ini
  [request_definition]
  r = sub, obj, act
  r2 = sub, obj, act
  [policy_definition]
  # 为何定义的两条策略定义只有第一条生效？
  p = sub, obj, act
  p2 = sub, sub2, obj, act
  [role_definition]
  g = _, _
  [policy_effect]
  e = some(where (p.eft == allow)) && !some(where (p.eft == deny))
  e2 = some(where (p.eft == allow)) && !some(where (p.eft == deny))
  [matchers]
  m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
  m2 = g(r2.sub, p2.sub) && g(r2.sub, p2.sub2) && r2.obj == p2.obj && r2.act == p2.act
  ```

  比如下面定义了两组 r p e m， 如果想使用 r2 p2 e2 m2 执行权限校验可以：

  ```java
  EnforceContext context = new EnforceContext("2");
  boolean ret4 = enforcer.enforce(context, "10005", "/sysA/config/bz1", "POST");
  ```

### ACL模型案例

model.conf

```properties
# Request definition
[request_definition]
# 某人（sub）以某方法访问（act）某资源（obj）
r = sub, act, obj
#r2 = sub, act
#r3 = sub, sub2, obj, act

# Policy definition
[policy_definition]
# 某人(sub)对某资源（obj）以某方法（act）访问，是否可以(eft,不写默认true)
p = sub, obj, act
#p2 = sub, act

# Policy effect，整合多个策略的结果
[policy_effect]
# 存在一个策略的结果为 allow, 就通过
e = some(where (p.eft == allow))
# 存在任何一个策略为allow,且不能有任何一个策略为deny
#e2 = some(where (p.eft == allow)) && !some(where (p.eft == deny))

# Matchers，请求和策略的匹配规则
[matchers]
# 发出请求的人必须与策略规定的主体相同，且请求的资源必须和策略规定的资源相同，且请求的动作必须和策略规定的动作相同
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

policy.csv，定义 policy_definition 中的策略集合。

```csv
p, alice, data1, read		# alice 可以读 data1, 对应 p = sub, obj, act，即 alice(sub) data1(obj) read(act)
p, bob, data2, write		# bob 可以写 data2
```

请求：

```
alice, read, data1			# alice 读 data1
```

### RBAC模型案例

model.conf

```properties
[request_definition]
# 某人/角色（sub）对某资源（obj）以某方法访问（act）
r = sub, obj, act

[policy_definition]
# 某人/角色(sub)对某资源（obj）以某方法（act）访问，是否可以(eft,不写默认true)
p = sub, obj, act

# 用于定义角色继承关系
[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
# g(r.sub, p.sub) 请求主体需要拥有策略中规定的主体，
m = g(r.sub, p.sub) && keyMatch(r.obj, p.obj) && (r.act == p.act || p.act == "*")
```

policy.csv

```
p, ROLE_USER, /data/users/*, *
p, ROLE_ADMIN, /data/users/*, *
p, ROLE_ADMIN, /data/admins/*, *
g, user, ROLE_USER					# 用户user继承ROLE_USER角色
g, admin, ROLE_USER
g, admin, ROLE_ADMIN
```

请求：

```
admin, /data/users/*，*
```

**向 Enforcer 提交请求后，会先通过 matchers 中的匹配器查找与请求匹配的策略规则（p），可能查找到的规则有多条且结果可能不同，最终由 policy_effect 综合多个规则的授权结果返回最终结果**。

## 工作原理简述

### 权限校验逻辑

JCasbin的访问控制（权限校验）逻辑全在 `Enforcer#enforce()` 这个方法中，还有一个批量校验的方法 `Enforcer#batchEnforce()`。

[understanding-casbin-detai](https://casbin.org/zh/docs/understanding-casbin-detail) 这里面给出了三张图，这些就是JCasbin权限校验的工作原理。

### 执行器类型

| 执行器                                                       | 作者   | 描述                                                         |
| ------------------------------------------------------------ | ------ | ------------------------------------------------------------ |
| [Enforcer](https://github.com/casbin/casbin/blob/master/enforcer.go) | Casbin | `Enforcer`是用户与Casbin策略和模型交互的基本结构。 你可以在[这里](https://casbin.org/zh/docs/management-api)找到有关`Enforcer` API的更多详细信息。 |
| [CachedEnforcer](https://github.com/casbin/casbin/blob/master/enforcer_cached.go) | Casbin | `CachedEnforcer`基于`Enforcer`，支持使用映射在内存中缓存请求的评估结果。 它提供了在指定过期时间内清除缓存的能力。 此外，它通过读写锁保证了线程安全。 你可以使用`EnableCache`来启用评估结果的缓存（默认为启用）。 `CachedEnforcer`的其他API方法与`Enforcer`相同。 |
| [DistributedEnforcer](https://github.com/casbin/casbin/blob/master/enforcer_distributed.go) | Casbin | `DistributedEnforcer`支持分布式集群中的多个实例。 它为调度器包装了`SyncedEnforcer`。 你可以在[这里](https://casbin.org/zh/docs/dispatchers#distributedenforcer)找到有关调度器的更多详细信息。 |
| [SyncedEnforcer](https://github.com/casbin/casbin/blob/master/enforcer_synced.go) | Casbin | `SyncedEnforcer`基于`Enforcer`，提供同步访问。 它是线程安全的。 |
| [SyncedCachedEnforcer](https://github.com/casbin/casbin/blob/master/enforcer_cached_synced.go) | Casbin | `SyncedCachedEnforcer`包装了`Enforcer`，提供决策同步缓存。   |

### 策略数据的存储策略

JCasbin 支持文件、JDBC、Hibernate、MyBatis、MongoDB、Redis等等的[适配器](https://casbin.org/zh/docs/adapters)。

注意：**JCasbin 的模型和策略数据会在 Enforcer 初始化时加载到内存，权限校验时并不会和数据库交互**，这样虽然性能很高但是需要在权限发生变动后提供额外的处理来保持内存中数据和数据源中的数据保持一致。

比如在管理页面添加、删除、修改策略数据后需要调用相关的API将变更的数据提交到内存，然后调用savePolicy() 将数据保存到数据库。

> 看有开源项目还额外建了一些表存储策略数据，可能嫌弃默认的表，不方便管理。

### Casbin执行器多实例之间的一致性

多个服务实例，每个实例可能包含一个或多个执行器实例，如何保证不同服务实例的执行器模型数据的一致性。

Casbin提供了两种方式：

+ Watchers: 借助分布式消息系统实现，比如 Etcd、Redis、Kafka。
+ Dispatchers: 调度器提供了一种同步策略增量更改的方式，约定基于一致性算法，如Raft，以确保所有执行器实例的一致性。 通过调度器，用户可以轻松建立分布式集群。

#### Redis Watcher

`casbin-spring-boot-starter` 默认内置了 [lettuce-redis-watcher](https://github.com/jcasbin/lettuce-redis-watcher)（仅仅400行代码），基于Lettuce客户端以及Redis的发布订阅实现，**所有执行器实例都订阅同一个主题，其中任意一个执行器实例上有修改策略数据就通过此主体发布事件，然后将数据同步到其他执行器实例**。

`casbin-spring-boot-starter` 将 Redis Watcher 的功能都封装好了，基本不需要自己再额外修改，使用上是无感的，比如调用 addPolicy() 等方法，它会自动发布事件给其他执行器，其他执行器也会自动进行同步。

不过貌似 addPolicy() 保存到数据库时是先清空数据库再保存，

不过如果想修改策略主题，可以通过 `casbin.policyTopic`修改。

> Lettuce是一个基于netty和Reactor的可扩展线程安全的Redis客户端。

开启 Redis Watcher 需要添加下面依赖和配置

依赖：

```xml
<-- 因为casbin-spring-boot-starter中这两个依赖的依赖范围是compileOnly，没有包含到starter的jar包中 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.casbin</groupId>
    <artifactId>jcasbin-redis-watcher</artifactId>
    <version>1.4.1</version>
</dependency>
```

配置：

```yml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    database: 0
    timeout: 6000
    lettuce:
      pool:
        enabled: true
        max-idle: 10
        max-wait: 3000
        min-idle: 0
        max-active: 8
        time-between-eviction-runs: 2000
```



## Spring Boot 整合

看 JCasbin 已经提供了 Spring Boot Starter，参考：[casbin-spring-boot-starter](https://github.com/jcasbin/casbin-spring-boot-starter)。

配置：

```yaml
casbin:
    # 是否启用 Casbin，默认为启用。
    enableCasbin: true
	# 是否使用同步 Enforcer，默认为 false。
    useSyncedEnforcer: false
	# 是否启用自动策略保存，如果适配器支持此功能，则默认启用。
    autoSave: true
    # 存储类型 [file, jdbc]，目前支持的 jdbc 数据库有 [mysql (mariadb), h2, oracle, postgresql, db2]。
    # 欢迎编写并提交您正在使用的 jdbc 适配器，参见：org.casbin.adapter.OracleAdapter
    # jdbc 适配器将主动查找您在 spring.datasource 中配置的数据源信息。
    # 默认使用 jdbc，并使用内置的 h2 数据库进行内存存储。
    storeType: jdbc
    # 当使用 jdbc 时自定义策略表名称，默认为 casbin_rule。
    tableName: casbin_rule
	# 数据源初始化策略 [create (自动创建数据表，如果已创建则不再初始化), never (始终不初始化)]
    initializeSchema: create
	# 本地模型配置文件地址，默认读取位置：classpath: casbin/model.conf
    model: classpath:casbin/model.conf
	# 如果在默认位置未找到模型配置文件并且 casbin.model 设置不正确，则使用内置的默认 rbac 模型，默认生效。
    useDefaultModelIfModelNotSetting: true
    # 本地策略配置文件地址，默认读取位置：classpath: casbin/policy.csv
    #如果在默认位置未找到配置文件，则抛出异常。
    #此配置项仅在 casbin.storeType 设置为 file 时生效。
    policy: classpath:casbin/policy.csv
    # 是否启用 CasbinWatcher 机制，默认不启用。
    # 如果启用该机制，casbin.storeType 必须为 jdbc，否则配置无效。
    enableWatcher: false
  	# CasbinWatcher 通知模式，默认使用 Redis 进行通知同步，暂时仅支持 Redis。
    # 在启用 Watcher 后，需要手动添加 spring-boot-starter-data-redis 依赖。
    watcherType: redis
    policyTopic: CASBIN_POLICY_TOPIC
    exception: 
    	# 当删除策略失败时抛出异常, 默认false
    	removePolicyFailed: false
```

### 结合官方案例分析

casbin-spring-boot-example 实现很简单；

看源码发现其实是将JCasbin整合到了Spring Security中；

其实完全没有必要必要引入 Spring Security, 原因看后面的分析。

```verilog
src
└── main
    ├── java
    │   └── org
    │       └── casbin
    │           ├── Application.java 	//@EnableGlobalMethodSecurity(securedEnabled = true)
										//开启 @Secured 权限校验
    │           ├── config
    │           │   ├── AuthenticationImpl.java	//自定义的简单身份认证信息类
    │           │   ├── CasbinFilter.java 	//通过Casbin实现的一个权限校验过滤器
    │           │   ├── GrantedAuthorityImpl.java 	//
    │           │   └── SecurityConfiguration.java	//@EnableWebSecurity；构造器注入Casbin的Enforcer、自定义的IUserService用户服务；将CasbinFilter注册到Spring Security的过滤器链，放在 BasicAuthenticationFilter之前；过滤器链匹配“/data/**”; 关闭了CSRF防御
    │           ├── controller
    │           │   ├── AuthController.java	//提供登录、注销接口 /auth/login、/auth/logout, 登录使用的cookie-session机制
    │           │   └── DataController.java	//提供了3个/data/**接口，通过 @Secured 检查用户角色进行权限校验
    │           ├── model
    │           │   ├── Data.java	//record类型
    │           │   └── User.java	//用户实体，包括用户名密码
    │           └── service
    │               ├── DataService.java
    │               ├── IDataService.java	//简单的带权限控制的服务
    │               ├── IUserService.java	
    │               └── UserService.java	//用Map存储了两个测试用户以及session信息
    └── resources
        ├── application.yml //casbin.storeType=file，即模型和策略数据存在文件中
        └── casbin
            ├── model.conf
            └── policy.csv
```

**工作流程**：

+ casbin-spring-boot-starter 中创建 Enforcer 实例
+ 注册 CasbinFilter 到 Spring Security 安全过滤器链，放 BasicAuthenticationFilter 前面（并不是说BasicAuthenticationFilter会被开启）；

+ /auth/login用户名密码登录，因为路径不匹配不会被安全过滤器拦截，所以直接进入 Controller，进行用户名密码校验，然后将用户信息存到session; 

+ 访问 /data/下接口，请求时带上 sessionId，请求会被安全过滤器拦截，经过**CasbinFilter#doFilter() 执行权限校验**`enforcer.enforce(user, path, method)`，校验通过后新建认证信息实例 AuthenticationImpl ，放到安全上下文，然后放行；

  ```java
  securityContext.setAuthentication(new AuthenticationImpl(u.get().getUsername(), rolesForUser));
  ```

+ FilterSecurityInterceptor 中借助 @Secured处理逻辑从 AuthenticationImpl 中获取角色信息做**二次校验**；

  这里可以看到其实做了重复的校验，Spring Security 这次校验完全没有必要。

**总结**：

由测试案例可以看到，关于 JCasbin 的集成，也仅仅只需要：

1、引入 `casbin-spring-boot-starter`；

2、选择以及配置模型策略存储方式；

3、注册一个请求过滤器或拦截器。

主要的工作量还是在于模型和策略的定义。

