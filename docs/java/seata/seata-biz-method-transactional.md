# [事务发起者和参与者服务方法上到底需不需要加 @Transactional 注解

Seata 初学时看一些 Seata Demo 产生的疑问（有些 Demo 有给参与者加 @Transactional 、有些 Demo 没有加 @Transactional，多数 Demo 都有加），看官方文档并没有说需要加 @Transational。

从源码看Seata并不强制依赖 @Transactional 开启的事务；即便发起者和参与者服务方法上都不添加 @Transactional 注解，Seata依然可以实现全局事务下所有分支事务的同步提交和回滚。

因为没有 @Transactional 注解时，Seata DataSourceProxy 会**为每个SQL写操作开启一个独立的分支事务并会上报给TC**，即没有 @Transactional 注解时，如果一个业务方法中有多个SQL写操作，执行这个业务方法会创建多个分支事务。
如果有 @Transactional 注解， Seata 会**使用 @Transactional 拦截器开启的事务作为分支事务，而不是为每个SQL创建一个分支事务**。

加不加 @Transactional 只是看这个业务方法中存在多个SQL写操作时，这多个SQL操作是否需要借助本地事务的 ACID 解决一些问题。

**如果一个业务方法中仅有一个SQL写操作是完全没有必要加 @Transactional 注解的**。

