# [ShardingSphere-JDBC](https://shardingsphere.apache.org/document/current/cn/overview/)

现在（5.4.1）已经改名为 `ShardingSphere-JDBC` 。

可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架。

ShardingSphere 另外两个产品：ShardingProxy 定位类似 MyCAT 用于服务端分库分表，ShardingSidecar 则是针对 service mesh 定位的一个分库分表插件。

**目标**：

+ 分库分表原理
+ 绑定表为何不会在联表查询时产生笛卡尔积
+ 读写分离的负载均衡规则
+ 分布式事务选择&各种实现原理
+ 怎么用 Antlr4 解析 SQL AST 并改写 SQL 的



## 源码编译

直接 mvn 编译会报错，因为依赖 Antlr4 需要它自动生成的一些代码文件。



## 基本使用

### 配置文件

看配置文件吧，全都有了。



## 参考资料

+ [ShardingSphere 核心原理精讲](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/ShardingSphere%20%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86%E7%B2%BE%E8%AE%B2-%E5%AE%8C) (版本比较老但是也有参考意义)
