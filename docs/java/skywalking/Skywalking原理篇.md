# Skywalking 原理篇

本篇分析源码版本 oap v9.3.0 skywalking-agent v8.14.0。

原理主要参考流程图 skywalking-workflow.drawio，这个文档只是对流程图的补充。

目标：

+ Agent 中封装了什么逻辑？什么时候触发抓取日志并上报的？
+ OAP平台如何接收上报的数据？怎么转存到ES？索引存储方式又是怎么样的？
+ UI层怎么读取链路追踪数据的？

> Skywalking 使用方式很多，插件很多，这里只分析应用篇中最常用的方式。比如：
>
> kafka作为数据传输通道的数据流向：
>
> 监控数据：springboot→agent→kafka→skywalkingOAP→es→kibana
> 日志数据：springboot→agent→filebeat→kafka→skywalkingOAP→es→kibana
>
> gRPC作为数据传输通道的数据流向：
>
> 监控数据和日志数据：springboot→agent→gRPC方式→skywalkingOAP→es→kibana



## SkyWalking Agent原理分析

> skywalking oap 选择的9.3.0版本，对应 skywalking-agent 选择8.14.0版本。
>
> 关于 javaagent 可以参考 byte-code jvm-sandbox-analysis 仓库，之前研究代理原理、Arthas、jvm-sandbox时曾有了解，也有简单地用过  Instrumentation agentmain() 和 Attach API。

SkyWalking 通过 Java Agent 在不侵入修改项目源码的情况下，从字节码层面（JVM层的AOP）给项目添加了追踪/监控/日志等能力。

给项目集成链路追踪能力时，需要通过 -javaagent 插桩 skywalking-agent.jar；源码地址：https://github.com/apache/skywalking-java/。其内部集成了很多插件和工具。

搜索源码可以看到它是从`apm-sniffer/apm-agent`模块下打包出来的。

```xml
<dependencies>
    <!--
	依赖 apm-agent-core 模块,  
	apm-agent-core 又依赖 java-agent-network java-agent-util java-agent-datacarrier 模块
	所以 skywalking-java.jar 是由这些模块打包生成的，像 apm-application-toolkit 中的组件、optional-plugins 等模块中插件并没有依赖也没打包进去
	-->
</dependencies>
<build>
        <finalName>skywalking-agent</finalName>
       	<plugins>
            <plugin>
                    <!--
                    maven-shade-plugin：打包
					maven-antrun-plugin：拷贝到项目根目录下 skywalking-agent 目录中
                    -->
            </plugin>
        </plugins>
</build>
```

> agent 核心代码就是上面几个模块，代码大概2W行，其他工具、插件占了7W多行代码。
>
> TODO: 工具组件、插件是怎么工作的？

代码入口是 premain() 方法，参考流程图中 Skywalking-Agent 工作原理 (v8.14.0)  。

### 嗅探配置初始化 SnifferconfigInitializer

其实就是将 config/agent.conf 、系统属性、-javaagent命令行属性写入 `org.apache.skywalking.apm.agent.core.conf.Config`类的各个内部类的 public static 成员变量中。

本地测试共加载了122个属性，`.`的前面是内部类名，后面是public static 成员变量的名字，只不过全部都转成了小写：

```verilog
AGENT_SETTINGS = {Properties@465}  size = 122
 "agent.keep_tracing" -> "false"
 "collector.get_agent_dynamic_config_interval" -> "20"
 "logging.resolver" -> "PATTERN"
 "collector.backend_service" -> "127.0.0.1:11800"
 "plugin.jedis.redis_parameter_max_length" -> "128"
 "log.max_message_size" -> "10485760"
 "plugin.micronauthttpserver.collect_http_params" -> "false"
 "logging.dir" -> ""
 "plugin.springmvc.collect_http_params" -> "false"
 "plugin.jdkthreading.threading_class_prefixes" -> ""
 "plugin.lettuce.redis_parameter_max_length" -> "128"
 "meter.active" -> "true"
 "plugin.springmvc.use_qualified_name_as_endpoint_name" -> "false"
 "agent.cluster" -> ""
 "logging.pattern" -> "%level %timestamp %thread %class : %msg %throwable"
 "statuscheck.max_recursive_depth" -> "1"
 "agent.enable" -> "true"
 "collector.is_resolve_dns_periodically" -> "false"
 "plugin.jedis.operation_mapping_read" -> "getrange,getbit,mget,hvals,hkeys,hlen,hexists,hget,hgetall,hmget,blpop,brpop,lindex,llen,lpop,lrange,rpop,scard,srandmember,spop,sscan,smove,zlexcount,zscore,zscan,zcard,zcount,xget,get,xread,xlen,xrange,xrevrange"
 "plugin.feign.filter_length_limit" -> "1024"
 "agent.operation_name_threshold" -> "150"
 "logging.output" -> "FILE"
 "plugin.kafka.topic_segment" -> "skywalking-segments"
 "plugin.kafka.get_topic_timeout" -> "10"
 "agent.ignore_suffix" -> ".jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.mp3,.mp4,.html,.svg"
 "plugin.guavacache.operation_mapping_read" -> "getIfPresent,get,getAllPresent,size"
 "plugin.toolkit.use_qualified_name_as_operation_name" -> "false"
 "plugin.solrj.trace_ops_params" -> "false"
 "plugin.neo4j.trace_cypher_parameters" -> "false"
 "plugin.springannotation.classname_match_regex" -> ""
 "plugin.kafka.topic_management" -> "skywalking-managements"
 "plugin.feign.supported_content_types_prefix" -> "application/json,text/"
 "agent.is_cache_enhanced_class" -> "false"
 "plugin.dubbo.provider_arguments_length_threshold" -> "256"
 "plugin.memcached.operation_mapping_write" -> "set,add,replace,append,prepend,cas,delete,touch,incr,decr"
 "profile.duration" -> "10"
 "buffer.buffer_size" -> "300"
 "agent.namespace" -> ""
 "plugin.toolkit.log.transmit_formatted" -> "true"
 "plugin.mongodb.trace_param" -> "false"
 "plugin.kafka.namespace" -> ""
 "logging.level" -> "INFO"
 "plugin.jedis.trace_redis_parameters" -> "false"
 "plugin.neo4j.cypher_body_max_length" -> "2048"
 "plugin.mongodb.filter_length_limit" -> "256"
 "plugin.cpupolicy.sample_cpu_usage_percent_limit" -> "-1"
 "plugin.mount" -> "plugins,activations"
 "collector.grpc_channel_check_interval" -> "30"
 "plugin.tomcat.collect_http_params" -> "false"
 "buffer.channel_size" -> "5"
 "agent.instance_properties_json" -> ""
 "plugin.kafka.bootstrap_servers" -> "localhost:9092"
 "correlation.value_max_length" -> "128"
 "plugin.exclude_plugins" -> ""
 "agent.service_name" -> "demo-application"
 "osinfo.ipv4_list_size" -> "10"
 "plugin.kafka.topic_profiling" -> "skywalking-profilings"
 "plugin.lettuce.trace_redis_parameters" -> "false"
 "plugin.neo4j.cypher_parameters_max_length" -> "512"
 "agent.ssl_trusted_ca_path" -> "/ca/ca.crt"
 "plugin.kafka.producer_config" -> ""
 "profile.dump_max_stack_depth" -> "500"
 "plugin.httpclient.collect_http_params" -> "false"
 "plugin.kafka.topic_meter" -> "skywalking-meters"
 "jvm.buffer_size" -> "600"
 "plugin.redisson.operation_mapping_read" -> "getrange,getbit,mget,hvals,hkeys,hlen,hexists,hget,hgetall,hmget,blpop,brpop,lindex,llen,lpop,lrange,rpop,scard,srandmember,spop,sscan,smove,zlexcount,zscore,zscan,zcard,zcount,xget,get,xread,xlen,xrange,xrevrange"
 "agent.is_open_debugging_class" -> "false"
 "plugin.dubbo.collect_consumer_arguments" -> "false"
 "agent.ssl_cert_chain_path" -> ""
 "agent.span_limit_per_segment" -> "2000"
 "meter.report_interval" -> "20"
 "plugin.solrj.trace_statement" -> "false"
 "plugin.redisson.trace_redis_parameters" -> "false"
 "collector.properties_report_period_factor" -> "10"
 "agent.ssl_key_path" -> ""
 "agent.trace_segment_ref_limit_per_span" -> "500"
 "agent.force_reconnection_period" -> "1"
 "plugin.kafka.topic_metrics" -> "skywalking-metrics"
 "plugin.influxdb.trace_influxql" -> "true"
 "plugin.jdbc.trace_sql_parameters" -> "false"
 "agent.cause_exception_depth" -> "5"
 "plugin.elasticsearch.trace_dsl" -> "false"
 "correlation.element_max_number" -> "3"
 "agent.authentication" -> ""
 "plugin.http.http_params_length_threshold" -> "1024"
 "plugin.http.http_headers_length_threshold" -> "2048"
 "logging.max_history_files" -> "-1"
 "profile.max_parallel" -> "5"
 "plugin.http.include_http_headers" -> ""
 "statuscheck.ignored_exceptions" -> ""
 "plugin.jedis.operation_mapping_write" -> "getset,set,setbit,setex,setnx,setrange,strlen,mset,msetnx,psetex,incr,incrby,incrbyfloat,decr,decrby,append,hmset,hset,hsetnx,hincrby,hincrbyfloat,hdel,rpoplpush,rpush,rpushx,lpush,lpushx,lrem,ltrim,lset,brpoplpush,linsert,sadd,sdiff,sdiffstore,sinterstore,sismember,srem,sunion,sunionstore,sinter,zadd,zincrby,zinterstore,zrange,zrangebylex,zrangebyscore,zrank,zrem,zremrangebylex,zremrangebyrank,zremrangebyscore,zrevrange,zrevrangebyscore,zrevrank,zunionstore,xadd,xdel,del,xtrim"
 "plugin.springtransaction.simplify_transaction_definition_name" -> "false"
 "plugin.kafka.topic_logging" -> "skywalking-logs"
 "profile.active" -> "true"
 "agent.class_cache_mode" -> "MEMORY"
 "plugin.redisson.redis_parameter_max_length" -> "128"
 "plugin.guavacache.operation_mapping_write" -> "put,putAll,invalidate,invalidateAll,invalidateAll,cleanUp"
 "plugin.dubbo.collect_provider_arguments" -> "false"
 "meter.max_meter_size" -> "500"
 "plugin.light4j.trace_handler_chain" -> "false"
 "agent.instance_name" -> ""
 "logging.max_file_size" -> "314572800"
 "plugin.jdbc.sql_parameters_max_length" -> "512"
 "collector.get_profile_task_interval" -> "20"
 "logging.file_name" -> "skywalking-api.log"
 "plugin.ehcache.operation_mapping_read" -> "get,getAll,getQuiet,getKeys,getKeysWithExpiryCheck,getKeysNoDuplicateCheck,releaseRead,tryRead,getWithLoader,getAll,loadAll,getAllWithLoader"
 "collector.heartbeat_period" -> "30"
 "plugin.jdbc.sql_body_max_length" -> "2048"
 "plugin.peer_max_length" -> "200"
 "agent.sample_n_per_3_secs" -> "-1"
 "agent.force_tls" -> "false"
 "plugin.memcached.operation_mapping_read" -> "get,gets,getAndTouch,getKeys,getKeysWithExpiryCheck,getKeysNoDuplicateCheck"
 "correlation.auto_tag_keys" -> ""
 "plugin.redisson.operation_mapping_write" -> "getset,set,setbit,setex,setnx,setrange,strlen,mset,msetnx,psetex,incr,incrby,incrbyfloat,decr,decrby,append,hmset,hset,hsetnx,hincrby,hincrbyfloat,hdel,rpoplpush,rpush,rpushx,lpush,lpushx,lrem,ltrim,lset,brpoplpush,linsert,sadd,sdiff,sdiffstore,sinterstore,sismember,srem,sunion,sunionstore,sinter,zadd,zincrby,zinterstore,zrange,zrangebylex,zrangebyscore,zrank,zrem,zremrangebylex,zremrangebyrank,zremrangebyscore,zrevrange,zrevrangebyscore,zrevrank,zunionstore,xadd,xdel,del,xtrim"
 "plugin.kafka.producer_config_json" -> ""
 "profile.snapshot_transport_buffer_size" -> "4500"
 "plugin.dubbo.consumer_arguments_length_threshold" -> "256"
 "collector.grpc_upstream_timeout" -> "30"
 "plugin.feign.collect_request_body" -> "false"
 "jvm.metrics_collect_period" -> "1"
 "plugin.micronauthttpclient.collect_http_params" -> "false"
 "plugin.ehcache.operation_mapping_write" -> "tryRemoveImmediately,remove,removeAndReturnElement,removeAll,removeQuiet,removeWithWriter,put,putAll,replace,removeQuiet,removeWithWriter,removeElement,removeAll,putWithWriter,putQuiet,putIfAbsent,putIfAbsent"
```

### 插件扫描与加载

插件定义文件，类加载器共扫描到126个：

```verilog
cfgUrlPaths = {ArrayList@1211}  size = 126
 0 = {URL@1273} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/graphql-16plus-plugin-8.14.0.jar!/skywalking-plugin.def"
 1 = {URL@1425} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-hbase-1.x-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 2 = {URL@1428} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mssql-jtds-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 3 = {URL@1432} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-sharding-sphere-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 4 = {URL@1433} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-quasar-plugin-8.14.0.jar!/skywalking-plugin.def"
 5 = {URL@1434} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/dubbo-3.x-conflict-patch-8.14.0.jar!/skywalking-plugin.def"
 6 = {URL@1435} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-cassandra-java-driver-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 7 = {URL@1436} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-activemq-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 8 = {URL@1437} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/baidu-brpc-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 9 = {URL@1438} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-elasticsearch-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 10 = {URL@1439} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-resttemplate-4.3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 11 = {URL@1440} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-hikaricp-3.x-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 12 = {URL@1441} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-httpasyncclient-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 13 = {URL@1442} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/nats-2.14.x-2.15.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 14 = {URL@1443} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-rabbitmq-plugin-8.14.0.jar!/skywalking-plugin.def"
 15 = {URL@1444} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/druid-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 16 = {URL@1445} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-xmemcached-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 17 = {URL@1446} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-influxdb-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 18 = {URL@1447} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/sofa-rpc-plugin-8.14.0.jar!/skywalking-plugin.def"
 19 = {URL@1448} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-finagle-6.25.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 20 = {URL@1449} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-lettuce-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 21 = {URL@1450} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-feign-default-http-9.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 22 = {URL@1451} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/dbcp-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 23 = {URL@1452} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-h2-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 24 = {URL@1453} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-nutz-mvc-annotation-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 25 = {URL@1454} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-impala-jdbc-2.6.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 26 = {URL@1455} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-hystrix-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 27 = {URL@1456} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-vertx-core-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 28 = {URL@1457} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-jetty-client-9.0-plugin-8.14.0.jar!/skywalking-plugin.def"
 29 = {URL@1458} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/okhttp-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 30 = {URL@1459} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-kafka-plugin-8.14.0.jar!/skywalking-plugin.def"
 31 = {URL@1460} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-armeria-0.85.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 32 = {URL@1461} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-shardingsphere-4.0.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 33 = {URL@1462} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-canal-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 34 = {URL@1463} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-asynchttpclient-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 35 = {URL@1464} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-light4j-plugin-8.14.0.jar!/skywalking-plugin.def"
 36 = {URL@1465} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-undertow-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 37 = {URL@1466} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-guava-eventbus-plugin-8.14.0.jar!/skywalking-plugin.def"
 38 = {URL@1467} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-springmvc-annotation-commons-8.14.0.jar!/skywalking-plugin.def"
 39 = {URL@1468} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mongodb-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 40 = {URL@1469} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mssql-jdbc-plugin-8.14.0.jar!/skywalking-plugin.def"
 41 = {URL@1470} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-elasticsearch-6.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 42 = {URL@1471} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/jedis-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 43 = {URL@1472} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-avro-plugin-8.14.0.jar!/skywalking-plugin.def"
 44 = {URL@1473} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-dubbo-plugin-8.14.0.jar!/skywalking-plugin.def"
 45 = {URL@1474} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mariadb-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 46 = {URL@1475} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-elasticjob-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 47 = {URL@1476} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-async-annotation-plugin-8.14.0.jar!/skywalking-plugin.def"
 48 = {URL@1477} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/jsonrpc4j-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 49 = {URL@1478} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-elasticsearch-7.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 50 = {URL@1479} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-redisson-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 51 = {URL@1480} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/graphql-9.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 52 = {URL@1481} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/dubbo-2.7.x-conflict-patch-8.14.0.jar!/skywalking-plugin.def"
 53 = {URL@1482} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-netty-socketio-plugin-8.14.0.jar!/skywalking-plugin.def"
 54 = {URL@1483} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-pulsar-2.8.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 55 = {URL@1484} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/tomcat-10x-plugin-8.14.0.jar!/skywalking-plugin.def"
 56 = {URL@1485} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-play-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 57 = {URL@1486} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-kafka-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 58 = {URL@1487} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-rocketmq-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 59 = {URL@1488} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mysql-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 60 = {URL@1489} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mysql-8.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 61 = {URL@1490} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-clickhouse-0.3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 62 = {URL@1491} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-nutz-http-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 63 = {URL@1492} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/graphql-12.x-15.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 64 = {URL@1493} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/spring-webflux-5.x-webclient-plugin-8.14.0.jar!/skywalking-plugin.def"
 65 = {URL@1494} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-struts2-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 66 = {URL@1495} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-concurrent-util-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 67 = {URL@1496} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-jetty-client-9.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 68 = {URL@1497} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-kylin-jdbc-2.6.x-3.x-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 69 = {URL@1498} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-okhttp-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 70 = {URL@1499} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-scheduled-annotation-plugin-8.14.0.jar!/skywalking-plugin.def"
 71 = {URL@1500} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/motan-plugin-8.14.0.jar!/skywalking-plugin.def"
 72 = {URL@1501} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-springmvc-annotation-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 73 = {URL@1502} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-rocketmq-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 74 = {URL@1503} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-jetty-server-9.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 75 = {URL@1504} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-dubbo-2.7.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 76 = {URL@1505} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/baidu-brpc-plugin-8.14.0.jar!/skywalking-plugin.def"
 77 = {URL@1506} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mongodb-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 78 = {URL@1507} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/resteasy-server-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 79 = {URL@1508} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/micronaut-http-client-plugin-8.14.0.jar!/skywalking-plugin.def"
 80 = {URL@1509} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-pulsar-2.2-2.7-plugin-8.14.0.jar!/skywalking-plugin.def"
 81 = {URL@1510} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-springmvc-annotation-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 82 = {URL@1511} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-xxl-job-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 83 = {URL@1512} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-core-patch-8.14.0.jar!/skywalking-plugin.def"
 84 = {URL@1513} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-springmvc-annotation-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 85 = {URL@1514} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-elastic-job-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 86 = {URL@1515} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-cloud-feign-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 87 = {URL@1516} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/thrift-plugin-8.14.0.jar!/skywalking-plugin.def"
 88 = {URL@1517} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mongodb-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 89 = {URL@1518} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-okhttp-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 90 = {URL@1519} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/dubbo-conflict-patch-8.14.0.jar!/skywalking-plugin.def"
 91 = {URL@1520} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-armeria-0.84.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 92 = {URL@1521} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-postgresql-8.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 93 = {URL@1522} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-cxf-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 94 = {URL@1523} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-undertow-worker-thread-pool-plugin-8.14.0.jar!/skywalking-plugin.def"
 95 = {URL@1524} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-tomcat-thread-pool-plugin-8.14.0.jar!/skywalking-plugin.def"
 96 = {URL@1525} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-shardingsphere-5.0.0-plugin-8.14.0.jar!/skywalking-plugin.def"
 97 = {URL@1526} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/graphql-8.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 98 = {URL@1527} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-sharding-sphere-4.1.0-plugin-8.14.0.jar!/skywalking-plugin.def"
 99 = {URL@1528} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-solrj-7.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 100 = {URL@1532} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/micronaut-http-server-plugin-8.14.0.jar!/skywalking-plugin.def"
 101 = {URL@1533} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-servicecomb-java-chassis-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 102 = {URL@1534} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-neo4j-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 103 = {URL@1535} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-httpclient-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 104 = {URL@1536} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-dubbo-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 105 = {URL@1537} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/tomcat-7.x-8.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 106 = {URL@1538} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-kafka-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 107 = {URL@1539} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-hutool-http-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 108 = {URL@1540} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-httpclient-5.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 109 = {URL@1541} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-mysql-6.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 110 = {URL@1542} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spring-cloud-feign-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 111 = {URL@1543} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/resteasy-server-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 112 = {URL@1544} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-spymemcached-2.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 113 = {URL@1545} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/jedis-2.x-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 114 = {URL@1546} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-grpc-1.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 115 = {URL@1547} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-httpClient-4.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 116 = {URL@1548} "jar:file:/home/lee/lib/skywalking/skywalking-agent/plugins/apm-vertx-core-3.x-plugin-8.14.0.jar!/skywalking-plugin.def"
 117 = {URL@1549} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-opentracing-activation-8.14.0.jar!/skywalking-plugin.def"
 118 = {URL@1550} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-logback-1.x-activation-8.14.0.jar!/skywalking-plugin.def"
 119 = {URL@1551} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-micrometer-activation-8.14.0.jar!/skywalking-plugin.def"
 120 = {URL@1552} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-trace-activation-8.14.0.jar!/skywalking-plugin.def"
 121 = {URL@1553} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-webflux-activation-8.14.0.jar!/skywalking-plugin.def"
 122 = {URL@1554} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-log4j-2.x-activation-8.14.0.jar!/skywalking-plugin.def"
 123 = {URL@1555} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-meter-activation-8.14.0.jar!/skywalking-plugin.def"
 124 = {URL@1556} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-log4j-1.x-activation-8.14.0.jar!/skywalking-plugin.def"
 125 = {URL@1557} "jar:file:/home/lee/lib/skywalking/skywalking-agent/activations/apm-toolkit-kafka-activation-8.14.0.jar!/skywalking-plugin.def"
```

以 skywalking-agent/plugins/apm-springmvc-annotation-5.x-plugin-8.14.0.jar!/skywalking-plugin.def 为例，看下其内容：

```properties
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.ControllerInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.RestControllerInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.HandlerMethodInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.reactive.InvocableHandlerMethodInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.reactive.ReactiveControllerInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.reactive.ReactiveRestControllerInstrumentation
```

### 字节码增强修改

#### JDK9 module

JDK9引入了模块这个新特性，将rt.jar拆成了很多小的模块，除了base模块，其他的都需要在module-info.java中指定依赖，然后才可以import 并调用。一方面可以减少引入不必要的代码，另外还可以增加代码安全性。

模块给 JavaAgent 带来了什么问题？为何要添加 JDK9ModuleExporter ？TODO

#### 增强执行时机

经过调试，可以看到是在类加载阶段执行了类增强（如果想弄清楚增强内部流程需要看ByteBuddy 和 Java Agent源码），以 RestControllerInstrumentation 为例。

mvc-annotation-5.x-plugin 插件定义（skywalking-plugin.def）：

```properties
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.ControllerInstrumentation
# 这里研究这个插件定义
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.RestControllerInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.HandlerMethodInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.reactive.InvocableHandlerMethodInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.reactive.ReactiveControllerInstrumentation
spring-mvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.reactive.ReactiveRestControllerInstrumentation
```

调用栈(只展示重要节点)：

```verilog

# agentBuilder.type(pluginFinder.buildMatch())
#                    .transform(new Transformer(pluginFinder))
#                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
#                    .with(new RedefinitionListener())
#                    .with(new Listener())
#                    .installOn(instrumentation);
transform:163, SkyWalkingAgent$Transformer (org.apache.skywalking.apm.agent)
doTransform:12104, AgentBuilder$Default$ExecutingTransformer (org.apache.skywalking.apm.dependencies.net.bytebuddy.agent.builder)
transform:12041, AgentBuilder$Default$ExecutingTransformer (org.apache.skywalking.apm.dependencies.net.bytebuddy.agent.builder)
...
transform:11950, AgentBuilder$Default$ExecutingTransformer (org.apache.skywalking.apm.dependencies.net.bytebuddy.agent.builder)
# Java 库标准接口 sun.instrument.TransformerManager 调用 ByteBuddy 
transform:188, TransformerManager (sun.instrument)
transform:428, InstrumentationImpl (sun.instrument)
defineClass1:-1, ClassLoader (java.lang)
...
findClass:362, URLClassLoader (java.net)
loadClass:418, ClassLoader (java.lang)
loadClass:355, Launcher$AppClassLoader (sun.misc)
...
forName:277, ClassUtils (org.springframework.util)
resolveBeanClass:456, AbstractBeanDefinition (org.springframework.beans.factory.support)
...
predictBeanType:649, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support)
isFactoryBean:1604, AbstractBeanFactory (org.springframework.beans.factory.support)
doGetBeanNamesForType:520, DefaultListableBeanFactory (org.springframework.beans.factory.support)
getBeanNamesForType:491, DefaultListableBeanFactory (org.springframework.beans.factory.support)
... 
processConfigurationClass:221, ConfigurationClassParser (org.springframework.context.annotation)
... # 其他Spring处理
refresh:532, AbstractApplicationContext (org.springframework.context.support)
...
refreshContext:397, SpringApplication (org.springframework.boot)
...
main:47, Application (cn.iocoder.springboot.lab39.skywalkingdemo)
```

### 启动所有BootService

```java
//ServiceManager
public void boot() {
    	//通过SPI机制加载所有BootService
    	//测试Demo中一共加载了20个
    	//0 = {TraceSegmentServiceClient@2274} 
        //1 = {ContextManager@2377} 
        //2 = {SamplingService@2378} 
        //3 = {GRPCChannelManager@2379} 
        //4 = {JVMMetricsSender@2380} 
        //5 = {JVMService@2381} 
        //6 = {ServiceManagementClient@2382} 
        //7 = {ContextManagerExtendService@2383} 
        //8 = {CommandService@2384} 
        //9 = {CommandExecutorService@2385} 
        //10 = {ProfileTaskChannelService@2386} 
        //11 = {ProfileSnapshotSender@2387} 
        //12 = {ProfileTaskExecutionService@2388} 
        //13 = {MeterService@2389} 
        //14 = {MeterSender@2390} 
        //15 = {StatusCheckService@2391} 
        //16 = {LogReportServiceClient@2392} 
        //17 = {ConfigurationDiscoveryService@2393} 
        //18 = {EventReportServiceClient@2394} 
        //19 = {ServiceInstanceGenerator@2395} 
        bootedServices = loadAllServices();

        prepare();
        startup();
        onComplete();
}
```

1. `TraceSegmentServiceClient`：负责将追踪数据上报到 SkyWalking 后端进行处理和存储。
2. `ContextManager`：用于管理上下文，存储和传递追踪数据和追踪上下文，是整个追踪系统的核心组件。
3. `SamplingService`：确定是否对某个追踪进行采样，以及如何采样，控制上报的采样率等。
4. `GRPCChannelManager`：管理 SkyWalking 的 gRPC 连接通道，提供连接池和自动重连功能。
5. `JVMMetricsSender`：收集 JVM 相关的度量指标，并将数据发送到 SkyWalking 后端进行处理和存储。
6. `JVMService`：提供 JVM 相关的信息，如内存、线程、CPU 等状态信息。
7. `ServiceManagementClient`：管理服务注册和发现，自动注册服务实例到 SkyWalking 服务注册中心。
8. `ContextManagerExtendService`：提供扩展的上下文管理服务，用于处理不同语言和框架的多样化场景。
9. `CommandService`：SkyWalking 的命令服务，可以向应用程序发送命令，如，重新启动 JVM。
10. `CommandExecutorService`：执行 SkyWalking 下发的命令。
11. `ProfileTaskChannelService`, `ProfileSnapshotSender`, `ProfileTaskExecutionService`：SkyWalking 的 profiler 相关组件，用于进行代码性能分析。
12. `MeterService`, `MeterSender`：Meter Service 提供度量采集，Meter Sender 则负责该采集数据的上报到 SkyWalking 后端。
13. `StatusCheckService`：检查和收集应用程序状态，例如数据库连接、MQ 机器运行状态等。
14. `LogReportServiceClient`：SkyWalking 支持对应用程序日志进行监控，该服务则负责收集和发送日志监控数据。
15. `ConfigurationDiscoveryService`：SkyWalking 动态配置服务，提供从远端配置中心获取配置的能力。
16. `EventReportServiceClient`：SkyWalking 事件报告服务，负责将应用程序中的事件上报到 SkyWalking 后端进行处理和存储。
17. `ServiceInstanceGenerator`：生成服务实例描述，反向探测服务实例状态等。

> 上面关于各 BootService 的功能描述是 ChatGPT 回答的，准确性未知，仅供参考。

### Agent 收集 Trace 数据

#### [OpenTracing 语义标准](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)

+ **Trace 调用链**

+ **Span** (跨度，对应一次方法调用、程序块调用、RPC/数据库访问)

  **包含的状态**：

  An operation name，操作名称
  A start timestamp，起始时间
  A finish timestamp，结束时间
  Span Tag，一组键值对构成的Span标签集合。键值对中，键必须为string，值可以是字符串，布尔，或者数字类型。
  Span Log，一组span的日志集合。 每次log操作包含一个键值对，以及一个时间戳。 键值对中，键必须为string，值可以是任意类型。 但是需要注意，不是所有的支持OpenTracing的Tracer,都需要支持所有的值类型。
  SpanContext，Span上下文对象 (下面会详细说明)
  References(Span间关系)，相关的零个或者多个Span（Span间通过SpanContext建立这种关系）

  **SpanContext 包含的状态：**

  任何一个OpenTracing的实现，都需要将当前调用链的状态（例如：trace和span的id），依赖一个独特的Span去跨进程边界传输
  Baggage Items，Trace的随行数据，是一个键值对集合，它存在于trace中，也需要跨进程边界传输

  **Span间关系：**

  OpenTracing目前定义了两种关系：ChildOf（父子） 和 FollowsFrom（跟随）。

#### SkyWalking Trace 模型

先看下Demo中抓取的一段 Trace 信息 (保存在 ContextManager CONTEXT, 类型TracingContext)：

```properties
result = {TracingContext@10266} 
     lastWarningTimestamp = 0
     # 记录某线程内部调用链路
     segment = {TraceSegment@10268} "TraceSegment{traceSegmentId='b327dabf42ab432fb69e6ac26b1c25b2.53.16896932079200000', ref=null, spans=[]}"
     activeSpanStack = {LinkedList@10269}  size = 1
          0 = {EntrySpan@10299} 						 #入口Span
               currentMaxDepth = 2							# 深度为何是2？看资料是说SpringMVC复用了在Tomcat拦截器中创建的EntrySpan, 然后栈的深度加1，
               stackDepth = 2
               peer = null
               spanId = 0
               parentSpanId = -1
               tags = {ArrayList@10362}  size = 2
               operationName = "GET:/demo/echo"
               layer = {SpanLayer@10304} "HTTP"
               isInAsyncMode = false
               isAsyncStopped = false
               owner = {TracingContext@10266} 
               startTime = 1689693207922
               endTime = 0
               errorOccurred = false
               componentId = 14
               logs = null
               refs = null
               skipAnalysis = false
     primaryEndpoint = {TracingContext$PrimaryEndpoint@10270} 
          span = {EntrySpan@10299} 
          this$0 = {TracingContext@10266} 
     spanIdGenerator = 1
     asyncSpanCounter = 0
     isRunningInAsyncMode = false
     asyncFinishLock = null
     running = true
     createTime = 1689693207921
     profileStatus = {ProfileStatusReference@10271} 
     correlationContext = {CorrelationContext@10272} 
     extensionContext = {ExtensionContext@10273} 
     spanLimitWatcher = {SpanLimitWatcher@10274} "AgentConfigChangeWatcher{propertyKey='agent.span_limit_per_segment'}"
```

借张图：

![](imgs/SkyWalking-Trace-Model.png)

+ **Distributed Trace** 

  一次分布式链路追踪（跨线程、跨进程的所有Segment的集合）。

+ **TraceSegment**

  用于记录某线程内部所有操作调用链路，一次分布式链路追踪可以包含多条 TraceSegment, 因为存在跨进程或跨线程、异步回调等。

  内部属性：

  + **traceSegmentId**  惟一ID， 格式：${应用实例ID(UUID)}.${线程ID}.${时间戳}*10000+线程自增序列([0, 9999]), 由 ``GlobalIdGenerator的`generate()` 生成

    比如上面的 `b327dabf42ab432fb69e6ac26b1c25b2.53.16896932079200000`。

  + **ref**  指向父TraceSegments

    通过父 traceSegmentId 和 spanId 属性，指向父级 TraceSegment 的指定 Span 。

    父 TraceSegment 应该是当前 TraceSegment 的来源。

  + **spans** 当前TraceSegment的span集合

    Span表示具体的某一个操作。

    内部属性：

    + spanId
    + parentSpanId
    + tags
    + operationName
    + layer 操作类型，DB 、RPC_FRAMEWORK 、HTTP 、MQ、CACHE
    + owner  此span位于的TracingContext
    + componentId 所属插件组件ID
    + refs 当前span所在的segment的前一个segment

    三种重要实现：

    + EntrySpan
    + LocalSpan
    + ExitSpan

  + relatedGlobalTraceId 当前Segment所属Trace的Id

  + ignore 是否忽略该条TraceSegment，即不收集链路信息

  + isSizeLimited 记录的Span是否超过上限，超过不再记录新的Span

  + createTime 创建时间戳

> 关于 SkyWalking Agent Trace 模型，详细参考：[SkyWalking 源码分析 —— Agent 收集 Trace 数据](https://www.iocoder.cn/SkyWalking/agent-collect-trace/)



### Agent 发送 Trace 数据

比如借助前面的 TraceSegmentServiceClient 将 Trace数据发送给OAP。





## OAP 接收数据并转储ES原理分析

### 接收 Agent 发送的 Trace 数据





## UI 层读取数据并展示原理



## 参考

+ [SkyWalking源码分析](https://www.iocoder.cn/categories/SkyWalking/)

