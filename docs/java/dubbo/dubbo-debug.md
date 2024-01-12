# Dubbo 源码调试

这里选择 3.2.9 版本，使用官方案例`dubbo-samples`测试。

Maven配置修改：

```xml
<!-- Dubbo版本 -->
<dubbo.version>3.2.9</dubbo.version>
<!-- 服务发现组件选择Nacos -->
<artifactId>dubbo-nacos-spring-boot-starter</artifactId>
```

application.yml

```yaml
dubbo:
  registry:
    # 使用Nacos作为注册中心
	address: nacos://${nacos.address:127.0.0.1}:8848
```

最好是能**将测试代码放到Dubbo源码中调试**，这样可以修改源码加入一些日志方便调试。

源码中虽然已经有了SpringBoot测试Demo, 但是不是用的Starter的配置方式。



## 源码工程中添加Demo

这里将`dubbo-samples`的SpringBoot示例代码放到源码`dubbo-demo`目录，修改POM文件，采用Starter方式引入依赖，使用Dubbo源码本地的Starter模块。

POM文件重点修改：

**父模块**：

只需要修改`dubbo-bom`的版本号，只需要和源码工程版本号一致，就会优先使用项目本地的`dubbo-bom`，模块位于 `dubbo-distribution/dubbo-bom`。

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-bom</artifactId>
    <!--<version>${dubbo.version}</version>-->
    <!--这样可以引入源码本地的dubbo-bom-->
    <version>${project.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

**Provider、Consumer模块**：

由于平时从Maven远程仓库下载Dubbo依赖，默认下载的是一个FatJar，包含了很多Dubbo模块，即会自动引入这些模块。而源码本地通过`dubbo-bom`还需要在子模块中明确指定要引入的模块。 

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-samples-spring-boot-interface</artifactId>
        <version>${project.parent.version}</version>
    </dependency>

    <!-- spring starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-actuator-autoconfigure</artifactId>
        <version>${spring-boot.version}</version>
    </dependency>

    <!-- dubbo -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <!--<artifactId>dubbo-zookeeper-curator5-spring-boot-starter</artifactId>-->
        <artifactId>dubbo-nacos-spring-boot-starter</artifactId>
    </dependency>
    
    <!--还需要手动添加 ====================================================================================== -->
    <!-- 注意 <version>${project.version}</version> 都可以删除，这里懒得一条一条地删了，不影响
 		可以根据依赖错误日志缺什么引入什么，
		也可以暴力地全部加进去，如果不清除依赖关系且感觉上面方式太麻烦的话
	-->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-cluster</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-common</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-compatible</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-compiler</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-config</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-config-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-config-spring</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-configcenter</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-configcenter-zookeeper</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-configcenter-apollo</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-configcenter-nacos</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-container</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-container-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-container-spring</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-dependencies</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-dependencies-zookeeper</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-dependencies-zookeeper-curator5</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-dependencies-bom</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-distribution</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-bom</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-filter</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-filter-cache</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-filter-validation</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-kubernetes</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata-report-zookeeper</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata-report-nacos</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata-report-redis</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata-processor</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metadata-definition-protobuf</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics-default</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics-registry</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics-prometheus</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics-metadata</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-metrics-config-center</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-monitor</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-monitor-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-monitor-default</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-native</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-native-plugin</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-maven-plugin</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-plugin</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-auth</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-security</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-qos-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-qos</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-reactive</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-security</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-multicast</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-multiple</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-nacos</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-zookeeper</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting-http</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting-netty</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting-netty4</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting-zookeeper</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-remoting-zookeeper-curator5</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-rpc</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-rpc-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-rpc-dubbo</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-rpc-injvm</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-rpc-rest</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-rpc-triple</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-serialization</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-serialization-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-serialization-hessian2</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-serialization-fastjson2</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-serialization-jdk</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot</artifactId>
        <version>${project.version}</version>
    </dependency>
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-actuator</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-autoconfigure</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-compatible</artifactId>
        <version>${project.version}</version>
    </dependency>
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-actuator-compatible</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-autoconfigure-compatible</artifactId>
        <version>${project.version}</version>
    </dependency>
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-dependencies-zookeeper</artifactId>
        <version>${project.version}</version>
        <type>pom</type>
    </dependency>
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-starters</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-observability-starters</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-observability-autoconfigure</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-tracing-otel-zipkin-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-tracing-otel-otlp-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-tracing-brave-zipkin-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-spring-boot-observability-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-nacos-spring-boot-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-zookeeper-spring-boot-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <!--<dependency>-->
    <!--    <groupId>org.apache.dubbo</groupId>-->
    <!--    <artifactId>dubbo-zookeeper-curator5-spring-boot-starter</artifactId>-->
    <!--    <version>${project.version}</version>-->
    <!--</dependency>-->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-check</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-common</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-modules</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-spring</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-spring3.2</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-spring4.1</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-test-spring4.2</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-xds</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```



