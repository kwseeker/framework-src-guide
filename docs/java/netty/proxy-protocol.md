# Proxy Protocol

[Proxy Protocol 规范](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)

[中文翻译](https://cloudnative.to/blog/proxy-protocol/)

Proxy Protocol 是 HAProxy 提供的一种代理协议，提供了一种方便的方式，可以安全地**传输连接信息**，例如客户端的地址，跨越多层 NAT 或 TCP 代理。它旨在对现有组件进行少量更改，并限制由传输信息处理引起的性能影响。

解决的主要痛点：通过代理中继 TCP 连接通常会导致原始 TCP 连接参数的丢失，例如源地址、目标地址、端口等等。

更多内容参考“背景”部分。

> 看到 RocketMQ Netty连接中有支持 Proxy Protocol，所以了解一下。
>
> 找到两篇文章，可以参考引入 Proxy Protocol 体验一下：
>
> [netty系列之:小白福利！手把手教你做一个简单的代理服务器](http://www.flydean.com/35-netty-simple-proxy/)
>
> [netty系列之:在netty中使用proxy protocol](https://cloud.tencent.com/developer/article/2170703)

