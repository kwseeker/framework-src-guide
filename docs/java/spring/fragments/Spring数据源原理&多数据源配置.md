# Spring 数据源原理&多数据源配置

> TODO: 看下源码。

Spring Boot配置数据源后，在启动时会自动扫描classpath下的所有jar包，如果发现了HikariCP、Tomcat、DBCP、DBCP2、C3P0等数据源连接池的依赖，就会自动配置数据源。如果没有发现这些依赖，就会使用Spring Boot默认的数据源连接池——Tomcat JDBC Pool。

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/test 
spring.datasource.username=root 
spring.datasource.password=123456 
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

多数据源配置：

主要是创建多个数据源实例，有时可能还需要创建一个动态数据源，用于将请求根据路由规则路由到不通的数据源；比如读写分离，写路由到主数据源，读路由到从数据源。

参考：

+ [85.2 Configure Two DataSources](https://docs.spring.io/spring-boot/docs/2.1.x/reference/html/howto-data-access.html#howto-two-datasources)
+ [配置多数据源](https://www.liaoxuefeng.com/article/1182502273240832)

