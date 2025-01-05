# Seata Server 源码启动配置

版本： v2.0.0。

**编译**：

```shell
mvn clean package -Dmaven.test.skip=true
```

源码编译可能发现 seata-console 有 npm 包下载失败，可以过滤掉这个模块:

```xml
<!--<module>console</module>-->
```

源码启动需要依赖 seata-serializer-protobuf下 protobuf 文件生成的代码，执行 `protobuf:compile` 生成代码；

最后启动服务端。

**配置**：

Seata Server 核心就是事务协调器，本身是 SpringBoot Web 应用，接入了服务发现、配置中心、事务持久化等功能，所以需要对这些功能进行选择性的配置；

原始配置：

默认都是用 JSON 文件。

```yaml
seata:
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: file
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: file
  store:
    # support: file 、 db 、 redis 、 raft
    mode: file
  #  server:
  #    service-port: 8091 #If not configured, the default is '${server.port} + 1000'
```

企业业务中常用配置：

1. 会话持久化配置

   常用数据库存储模式（db）比如： MySQL 数据库，Seata 2.0.0 默认使用 mysql-connector-java:5.1.42, 如果需要使用 MySQL8 存储会话，需要修改 `seata-server` 模块中依赖配置 `<version>8.0.27</version>`。

   ```yaml
   seata:
     store:
       # support: file 、 db 、 redis 、 raft
       mode: db
       db:
         driverClassName: com.mysql.cj.jdbc.Driver
         url: jdbc:mysql://127.0.0.1:13306/seata?useSSL=false&useUnicode=true&characterEncoding=UTF-8
         user: root
         password: 123456
   ```

   关于数据库详细配置参考 `StoreDBProperties`。

   使用MySQL数据库存储会话，需要创建数据库和表，SQL参考源码根目录 script 目录。

   ```sql
   -- -------------------------------- The script used when storeMode is 'db' --------------------------------
   -- the table to store GlobalSession data
   CREATE TABLE IF NOT EXISTS `global_table`
   (
       `xid`                       VARCHAR(128) NOT NULL,
       `transaction_id`            BIGINT,
       `status`                    TINYINT      NOT NULL,
       `application_id`            VARCHAR(32),
       `transaction_service_group` VARCHAR(32),
       `transaction_name`          VARCHAR(128),
       `timeout`                   INT,
       `begin_time`                BIGINT,
       `application_data`          VARCHAR(2000),
       `gmt_create`                DATETIME,
       `gmt_modified`              DATETIME,
       PRIMARY KEY (`xid`),
       KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
       KEY `idx_transaction_id` (`transaction_id`)
   ) ENGINE = InnoDB
     DEFAULT CHARSET = utf8mb4;
   
   -- the table to store BranchSession data
   CREATE TABLE IF NOT EXISTS `branch_table`
   (
       `branch_id`         BIGINT       NOT NULL,
       `xid`               VARCHAR(128) NOT NULL,
       `transaction_id`    BIGINT,
       `resource_group_id` VARCHAR(32),
       `resource_id`       VARCHAR(256),
       `branch_type`       VARCHAR(8),
       `status`            TINYINT,
       `client_id`         VARCHAR(64),
       `application_data`  VARCHAR(2000),
       `gmt_create`        DATETIME(6),
       `gmt_modified`      DATETIME(6),
       PRIMARY KEY (`branch_id`),
       KEY `idx_xid` (`xid`)
   ) ENGINE = InnoDB
     DEFAULT CHARSET = utf8mb4;
   
   -- the table to store lock data
   CREATE TABLE IF NOT EXISTS `lock_table`
   (
       `row_key`        VARCHAR(128) NOT NULL,
       `xid`            VARCHAR(128),
       `transaction_id` BIGINT,
       `branch_id`      BIGINT       NOT NULL,
       `resource_id`    VARCHAR(256),
       `table_name`     VARCHAR(32),
       `pk`             VARCHAR(36),
       `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
       `gmt_create`     DATETIME,
       `gmt_modified`   DATETIME,
       PRIMARY KEY (`row_key`),
       KEY `idx_status` (`status`),
       KEY `idx_branch_id` (`branch_id`),
       KEY `idx_xid` (`xid`)
   ) ENGINE = InnoDB
     DEFAULT CHARSET = utf8mb4;
   
   CREATE TABLE IF NOT EXISTS `distributed_lock`
   (
       `lock_key`       CHAR(20) NOT NULL,
       `lock_value`     VARCHAR(20) NOT NULL,
       `expire`         BIGINT,
       primary key (`lock_key`)
   ) ENGINE = InnoDB
     DEFAULT CHARSET = utf8mb4;
   
   INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
   INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
   INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
   INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
   ```

2. 配置中心和注册中心

   线上常用 Nacos 等中间件作为配置中心和注册中心。

3. 源码调试配置

   源码中有很多超时处理，如果想要单步调试，需要修改这些超时时间避免报超时异常，或者就将测试应用放到Seata工程中通过打日志调试。

   服务端：

   ```yaml
   seata:
     transport:
       rpc-rm-request-timeout: 54000
       rpc-tm-request-timeout: 108000
       rpc-tc-request-timeout: 54000
     server:
       distributedLockExpireTime: 36000000
   ```

   客户端：

   ```yaml
   seata:
     tm:
       defaultGlobalTransactionTimeout: 60000
   ```

**日志：**

一次成功的分布式事务执行：

```verilog
# 首先是客户端 RM TM 的注册
15:11:34.966  INFO --- [ttyServerNIOWorker_1_1_16] [rocessor.server.RegTmProcessor] [      onRegTmMessage]  [] : TM register success,message:RegisterTMRequest{version='2.0.0', applicationId='account-service', transactionServiceGroup='account-service-group', extraData='ak=null
digest=account-service-group,172.17.0.1,1735888294827
timestamp=1735888294827
authVersion=V4
vgroup=account-service-group
ip=172.17.0.1
'},channel:[id: 0x98943862, L:/127.0.0.1:8091 - R:/127.0.0.1:42568],client version:2.0.0
15:11:36.032  INFO --- [rverHandlerThread_1_1_500] [rocessor.server.RegRmProcessor] [      onRegRmMessage]  [] : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://127.0.0.1:13306/seata_account', version='2.0.0', applicationId='account-service', transactionServiceGroup='account-service-group', extraData='null'},channel:[id: 0xac236558, L:/127.0.0.1:8091 - R:/127.0.0.1:42580],client version:2.0.0
15:11:40.581  INFO --- [ttyServerNIOWorker_1_3_16] [rocessor.server.RegTmProcessor] [      onRegTmMessage]  [] : TM register success,message:RegisterTMRequest{version='2.0.0', applicationId='product-service', transactionServiceGroup='product-service-group', extraData='ak=null
digest=product-service-group,172.17.0.1,1735888300483
timestamp=1735888300483
authVersion=V4
vgroup=product-service-group
ip=172.17.0.1
'},channel:[id: 0x658d242e, L:/127.0.0.1:8091 - R:/127.0.0.1:38128],client version:2.0.0
15:11:41.277  INFO --- [rverHandlerThread_1_2_500] [rocessor.server.RegRmProcessor] [      onRegRmMessage]  [] : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://127.0.0.1:13306/seata_product', version='2.0.0', applicationId='product-service', transactionServiceGroup='product-service-group', extraData='null'},channel:[id: 0x9df205ee, L:/127.0.0.1:8091 - R:/127.0.0.1:38138],client version:2.0.0
15:11:44.428  INFO --- [ttyServerNIOWorker_1_5_16] [rocessor.server.RegTmProcessor] [      onRegTmMessage]  [] : TM register success,message:RegisterTMRequest{version='2.0.0', applicationId='order-service', transactionServiceGroup='order-service-group', extraData='ak=null
digest=order-service-group,172.17.0.1,1735888304356
timestamp=1735888304356
authVersion=V4
vgroup=order-service-group
ip=172.17.0.1
'},channel:[id: 0x4e800afc, L:/127.0.0.1:8091 - R:/127.0.0.1:38140],client version:2.0.0
15:11:45.004  INFO --- [rverHandlerThread_1_3_500] [rocessor.server.RegRmProcessor] [      onRegRmMessage]  [] : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://127.0.0.1:13306/seata_order', version='2.0.0', applicationId='order-service', transactionServiceGroup='order-service-group', extraData='null'},channel:[id: 0x59526444, L:/127.0.0.1:8091 - R:/127.0.0.1:38142],client version:2.0.0
15:12:22.986  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='createOrder(java.lang.Long, java.lang.Long, java.lang.Integer)', timeout=60000000}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:23.011  INFO --- [rverHandlerThread_1_4_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [172.17.0.1:8091:7260406562673074177] : Begin new global transaction applicationId: order-service,transactionServiceGroup: order-service-group, transactionName: createOrder(java.lang.Long, java.lang.Long, java.lang.Integer),timeout:60000000,xid:172.17.0.1:8091:7260406562673074177
15:12:23.012  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='172.17.0.1:8091:7260406562673074177', extraData='null', resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:23.616  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='172.17.0.1:8091:7260406562673074177', branchType=AT, resourceId='jdbc:mysql://127.0.0.1:13306/seata_product', lockKey='product:1', applicationData='{"autoCommit":false}'}, clientIp: 127.0.0.1, vgroup: product-service-group
15:12:23.728  INFO --- [nPool.commonPool-worker-2] [.jraft.util.JRaftServiceLoader] [         newProvider]  [172.17.0.1:8091:7260406562673074177] : SPI service [com.alipay.sofa.jraft.rpc.RaftRpcFactory - com.alipay.sofa.jraft.rpc.impl.BoltRaftRpcFactory] loading.
Sofa-Middleware-Log SLF4J : Actual binding is of type [ com.alipay.remoting Logback ]
15:12:23.832  INFO --- [nPool.commonPool-worker-2] [com.alipay.sofa.common.log    ] [              report]  [172.17.0.1:8091:7260406562673074177] : Sofa-Middleware-Log SLF4J : Actual binding is of type [ com.alipay.remoting Logback ]
15:12:23.944  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [172.17.0.1:8091:7260406562673074177] : Register branch successfully, xid = 172.17.0.1:8091:7260406562673074177, branchId = 7260406562673074180, resourceId = jdbc:mysql://127.0.0.1:13306/seata_product ,lockKeys = product:1
15:12:23.944  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=7260406562673074180, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: product-service-group
15:12:24.371  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='172.17.0.1:8091:7260406562673074177', branchType=AT, resourceId='jdbc:mysql://127.0.0.1:13306/seata_account', lockKey='account:1', applicationData='{"autoCommit":false}'}, clientIp: 127.0.0.1, vgroup: account-service-group
15:12:24.393  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [172.17.0.1:8091:7260406562673074177] : Register branch successfully, xid = 172.17.0.1:8091:7260406562673074177, branchId = 7260406562673074182, resourceId = jdbc:mysql://127.0.0.1:13306/seata_account ,lockKeys = account:1
15:12:24.394  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=7260406562673074182, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: account-service-group
15:12:24.678  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='172.17.0.1:8091:7260406562673074177', branchType=AT, resourceId='jdbc:mysql://127.0.0.1:13306/seata_order', lockKey='orders:2', applicationData='{"skipCheckLock":true}'}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:24.687  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [172.17.0.1:8091:7260406562673074177] : Register branch successfully, xid = 172.17.0.1:8091:7260406562673074177, branchId = 7260406562673074185, resourceId = jdbc:mysql://127.0.0.1:13306/seata_order ,lockKeys = orders:2
15:12:24.687  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=7260406562673074185, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:24.736  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalCommitRequest{xid='172.17.0.1:8091:7260406562673074177', extraData='null'}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:24.752  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalCommitResponse{globalStatus=Committed, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:25.497  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='172.17.0.1:8091:7260406562673074177', branchId=7260406562673074180, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: product-service-group
15:12:25.502  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [172.17.0.1:8091:7260406562673074177] : Commit branch transaction successfully, xid = 172.17.0.1:8091:7260406562673074177 branchId = 7260406562673074180
15:12:25.509  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='172.17.0.1:8091:7260406562673074177', branchId=7260406562673074182, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: account-service-group
15:12:25.516  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [172.17.0.1:8091:7260406562673074177] : Commit branch transaction successfully, xid = 172.17.0.1:8091:7260406562673074177 branchId = 7260406562673074182
15:12:25.527  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='172.17.0.1:8091:7260406562673074177', branchId=7260406562673074185, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 127.0.0.1, vgroup: order-service-group
15:12:25.532  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [172.17.0.1:8091:7260406562673074177] : Commit branch transaction successfully, xid = 172.17.0.1:8091:7260406562673074177 branchId = 7260406562673074185
15:12:25.544  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [      doGlobalCommit]  [172.17.0.1:8091:7260406562673074177] : Committing global transaction is successfully done, xid = 172.17.0.1:8091:7260406562673074177.
```





