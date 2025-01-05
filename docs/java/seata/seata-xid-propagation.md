# 全局事务ID是怎么传递给事务参与者服务的

全局事务发起者调用参与者接口执行分支事务时并没有看到有传递XID，但是分支事务执行时 RootContext 却保存了正确的 XID，是怎么传递的？

其实调用接口时是有传XID的，以 HTTP 请求为例是通过请求头参数 `TX_XID` 传递的，只是设置这个请求头参数的代码略隐秘（不仔细一行行地看源码可能漏过去），在 `AbstractHttpExecutor#wrapHttpExecute()` 方法中在发送请求前为请求自动添加XID头信息；

```
headers.put(RootContext.KEY_XID, xid);
```

参与者服务通过拦截器 **TransactionPropagationInterceptor** 解析XID并保存到线程上下文；TransactionPropagationInterceptor 中的事务传播和 Spring 事务传播不是一个概念。

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    String rpcXid = request.getHeader(RootContext.KEY_XID);
    return this.bindXid(rpcXid);
}
```

除了 HTTP 其他几种服务调用方式应该也有对应的处理，后面再看。
