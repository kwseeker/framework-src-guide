# quarkus-mybatis

repo: https://github.com/quarkiverse/quarkus-mybatis

docs: https://docs.quarkiverse.io/quarkus-mybatis/dev/index.html

基本使用看官方文档即可，这里仅仅分析介绍不太详细的功能的使用。



## 事务控制

虽然文档中介绍的不清晰，但是源码中集成测试中有使用 Demo。

启用事务本质就是在SQL操作前后封装上事务管理操作的拦截器，quarkus-mybatis 依赖 **quarkus-narayana-jta** 实现事务功能，后者实现了 @Transactional 注解支持，还提供了 TransactionManager 的实现。

可以调试 quarkus-labs/quarkus-131-orm-mybatis 查看内部控制流程。
用到 quarkus-narayana-jta **NotifyingTransactionManager** 事务管理器和 quarkus-mybatis **TransactionalSqlSession**。

TODO: 内部封装原理与完整流程。

