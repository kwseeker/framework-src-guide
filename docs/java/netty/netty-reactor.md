# Netty Reactor 线程模型

Netty可以通过参数配置自动在Reactor单线程模型、Reactor多线程模型、Reactor主从多线程模型中切换，项目中多用**Reactor主从多线程模型**，通常会配置3组线程池，第一组用于处理 Accept 请求，第二组用于 Selector 监听数据就绪的通道并分发，第三组用于 Pipeline 中 ChannelHandler 链的执行对请求进行处理。

RocketMQ 中 NettyRemotingServer 就是Reactor主从多线程模型。

```java
// 这里分别配置了用于 Acceptor Selector 的线程池（eventLoopGroupBoss、eventLoopGroupSelector）
serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
    .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
    .option(ChannelOption.SO_BACKLOG, 1024)
    .option(ChannelOption.SO_REUSEADDR, true)
    .childOption(ChannelOption.SO_KEEPALIVE, false)
    .childOption(ChannelOption.TCP_NODELAY, true)
    .localAddress(new InetSocketAddress(this.nettyServerConfig.getBindAddress(),
                                        this.nettyServerConfig.getListenPort()))
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) {
            configChannel(ch);
        }
    });

protected ChannelPipeline configChannel(SocketChannel ch) {
    // Pipeline 中的 ChannelHandler 使用 defaultEventExecutorGroup 线程池
    return ch.pipeline()
        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, new HandshakeHandler())
        .addLast(defaultEventExecutorGroup,
                 encoder,
                 new NettyDecoder(),
                 distributionHandler,
                 new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                 connectionManageHandler,
                 serverHandler
                );
}
```

