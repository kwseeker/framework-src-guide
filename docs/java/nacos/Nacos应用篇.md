# Nacos 应用篇

[Nacos docs](https://nacos.io/zh-cn/docs/what-is-nacos.html)

## 引入Nacos注册中心

1. **安装Nacos Server服务并启动**

   下载启动参考: [Nacos快速开始](https://nacos.io/zh-cn/docs/quick-start.html)

   ```shell
   # 下载后解压到/opt
   sudo ln -s nacos-2.0.3 nacos
   vi ~/.zshrc
       # Nacos-2.0.3
       export NACOS_HOME=/opt/nacos
       export PATH=$NACOS_HOME/bin:$PATH
   source ~/.zshrc
   
   echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
   ```

   启动&关闭

   ```shell
   # 启动时可能遇到：ErrCode:500, ErrMsg:User limit of inotify watches reached Caused by: java.io.IOException: User limit of inotify watches reached
   #  inotify是一个内核用于通知用户空间程序文件系统变化的机制，如文件增加、删除等事件可以立刻让用户态得知
   cat /proc/sys/fs/inotify/max_user_watches
   echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
   # 单机模式
   sh startup.sh -m standalone
   # 集群模式
   
   # 启动成功日志（默认监听8848）
   # /opt/nacos/logs/start.out
   2021-09-05 12:18:26,284 INFO Exposing 16 endpoint(s) beneath base path '/actuator'
   2021-09-05 12:18:26,374 INFO Tomcat started on port(s): 8848 (http) with context path '/nacos'
   2021-09-05 12:18:26,377 INFO Nacos started successfully in stand alone mode. use embedded storage
   # 检查
   curl -X GET 'http://127.0.0.1:8848/nacos/actuator'
   # 关闭
   sh shutdown.sh
   ```
   

docker容器启动

```shell
docker pull nacos/nacos-server:v2.2.0  
# 使用v2版本如果使用
docker run --name nacos-single -e MODE=standalone -p 8848:8848 -p 9848:9848  -d nacos/nacos-server:v2.2.0
```

[Nacos 控制台](https://nacos.io/zh-cn/docs/console-guide.html) [登录页面](http://localhost:8848/nacos/index.html#/login)

[Nacos OpenAPI](https://nacos.io/zh-cn/docs/open-api.html)

2. **微服务引入Nacos客户端**

   参考：[Nacos dicovery](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery)

   **服务注册&发现接口**

   ```shell
   curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
   curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
   ```

   **引入 Nacos Discovery流程: **

   **a) 引入 Nacos Discovery Starter**

   ```xml
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
   </dependency>
   ```

   **b) 配置 Nacos 服务端**

   修改启动参数等。

   **c) 配置Nacos Discovery 客户端**

   业务中的微服务全部作为Nacos的客户端

   ```yaml
   spring:
     # Nacos 注册中心地址
     cloud:
       nacos:
         # 更多配置参考：NacosDiscoveryProperties.java
         # 比如：配置权限（username, password）、分组等
         discovery:
         　server-addr: 127.0.0.1:8848
           group: web-service
   ```

   **d) Nacos负载均衡器**

   将provider、consumer微服务注册到注册中心后，comsumer想要调用provider接口还要能够找到provider。怎么找就需要借助Nacos负载均衡器。

   拓展了Spring RestTemplate 的 ClientHttpRequestInterceptor。在这个拦截器里面依赖Netflix Ribbon组件引入了负载均衡策略。

   Ribbon中通过LoadBalancerInterceptor实现了负载均衡策略。

   consumer端只需要@LoadBalanced注解就可以引入Ribbon的负载均衡拦截器。

   ```java
   @Bean
   @LoadBalanced
   public RestTemplate restTemplate() {
       return new RestTemplate();
       //@LoadBalaced注解效果等同于
   	//RestTemplate restTemplate = new RestTemplate();
   	//restTemplate.setInterceptors(Collections.singletonList(new LoadBalancerInterceptor(loadBalancer)));
   	//return restTemplate;
   }
   ```

   > 负载均衡本质就是将注册的Provider服务节点转换为某实例的域名和端口。

   **服务注册成功之后详情数据**：

   ```json
   //http://localhost:8848/nacos/v1/ns/instance/list?serviceName=mall-order
   {
     "name": "DEFAULT_GROUP@@mall-order",
     "groupName": "DEFAULT_GROUP",
     "clusters": "",
     "cacheMillis": 10000,
     "hosts": [
       {
         //ip#port#集群名#GROUP名#服务名
         "instanceId": "192.168.2.169#8020#SH#DEFAULT_GROUP@@mall-order",
         "ip": "192.168.2.169",
         "port": 8020,
         "weight": 1.0,
         "healthy": true,
         "enabled": true,
         //是否是临时节点
         "ephemeral": false,
         "clusterName": "SH",
         "serviceName": "DEFAULT_GROUP@@mall-order",
         "metadata": {
           "preserved.register.source": "SPRING_CLOUD"
         },
         //心跳间隔
         "instanceHeartBeatInterval": 5000,
         "instanceHeartBeatTimeOut": 15000,
         "ipDeleteTimeout": 30000
       }
     ],
     "lastRefTime": 1654443462168,
     "checksum": "",
     "allIPs": false,
     "reachProtectionThreshold": false,
     "valid": true
   }
   ```

3. **微服务启动异常问题**

   ```txt
   com.alibaba.nacos.core.distributed.raft.exception.NoLeaderException: The Raft Group [naming_instance_metadata] did not find the Leader node;
   ```

   **原因**：Nacos 采用 raft 算法来计算 Leader，并且会记录前一次启动的集群地址，所以当我们自己的服务器 IP 改变时会导致 raft 记录的集群地址失效（看上面详情数据："ip": "192.168.2.169"），导致选 Leader 出现问题，只要删除 Nacos 根目录下 data 文件夹下的 protocol 文件夹即可。（TODO: 看源码时再看看这个原理？）

   

## 引入Nacos配置中心

