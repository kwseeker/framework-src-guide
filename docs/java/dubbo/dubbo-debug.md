# Dubbo 调试

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

