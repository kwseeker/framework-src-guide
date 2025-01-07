# @GlobalTransactional 注释的方法明明没有像 @Transactional 拓展事务操作为何这个方法中执行的SQL还是可以在出现异常后被回滚

比如下面代码：

分析源码可知 @GlobalTransactional 注解其实就是为方法增强了 `AdapterSpringSeataInterceptor` 这个拦截器中的流程，但是这个拦截器中根本没有事务操作，为什么在下面代码中保存完订单后出现异常还是可以实现订单回滚？

```java
@GlobalTransactional
public Integer createOrder(Long userId, Long productId, Integer price) throws Exception {
    Integer amount = 1; // 购买数量，暂时设置为 1。

    logger.info("[createOrder] 当前 XID: {}", RootContext.getXID());

    // 扣减库存
    this.reduceStock(productId, amount);

    // 扣减余额
    this.reduceBalance(userId, price);

    // 保存订单,
    // 保存订单操作不需要封装到本地事务中执行, 保存订单操作完成后返回前如果出现异常也是可以回滚的
    OrderDO order = new OrderDO().setUserId(userId).setProductId(productId).setPayAmount(amount * price);
    orderDao.saveOrder(order); // 因为内部经过 ConnectionProxy 实现本地事务操作，本地事务执行成功的话会向TC注册分支事务，记录UndoLog,然后执行本地事务提交
    logger.info("[createOrder] 保存订单: {}", order.getId());
    int i = 1 / 0;  // 模拟异常，异常发生后上面保存的订单同样是TC通知订单服务RM进行分支事务回滚的（订单服务TM先请求TC回滚所有分支事务，TC向所有服务RM（包括订单服务）发送分支事务回滚通知，订单服务RM接收到分支事务回滚通知，查询并执行UndoLog回滚保存的订单）

    // 返回订单编号
    return order.getId();
}
```

其实是借助 ConnectionProxy 执行事务操作的；详细参考 `AbstractDMLBaseExecutor#executeAutoCommitFalse(Object[] args)` 方法；

saveOrder() 最终是需要借助连接执行 SQL语句的，Seata 对连接进行了增强（ConnectionProxy，通过装饰器模式增强的不是AOP），会经过 executeAutoCommitFalse() 这个方法，在这个方法里面会先开启本地事务（关闭自动提交），然后执行SQL语句，如果执行有异常就回滚（通过 ConnectionProxy 回滚）没异常就提交（通过 ConnectionProxy 提交），最后恢复自动提交；

注意**这里提交的时候借助 ConnectionProxy#commit() 提交，而 ConnectionProxy#commit() 中会先判断当前线程上下文是否包含全局事务，是的话会先向 TC 注册一个分支事务，持久化 UndoLog，然后执行本地事务的提交**；

**异常发生后上面保存的订单同样是 TC 通知订单服务 RM 进行分支事务回滚的（订单服务 TM 先请求 TC 回滚所有分支事务，TC 向所有服务 RM（包括订单服务）发送分支事务回滚通知，订单服务 RM 接收到分支事务回滚通知，查询并执行UndoLog回滚保存的订单）**。