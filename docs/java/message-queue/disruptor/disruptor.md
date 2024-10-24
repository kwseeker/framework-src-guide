# [Disruptor](https://lmax-exchange.github.io/disruptor/)

LMAX Disruptor 的定位是 线程间高性能的消息库。

## 介绍

Disruptor是一个提供并发环形缓冲区数据结构的库。它旨在异步事件处理架构中提供低延迟、高吞吐量的工作队列。

**关键特性**：

+ 使用[消费者依赖关系图](https://lmax-exchange.github.io/disruptor/user-guide/index.html#_consumer_dependency_graph)将事件多播给消费者
+ 为事件[预分配内存](https://lmax-exchange.github.io/disruptor/user-guide/index.html#_event_pre_allocation)
+ [可选无锁](https://lmax-exchange.github.io/disruptor/user-guide/index.html#_optionally_lock_free)

**核心概念&工作原理简图**：

![](../imgs/disruptor-models.png)

> 官方文档中几行文字根本说不清，需要结合后面的源码分析每个组件的作用。

+ Ring Buffer
+ Sequence
+ Sequencer
+ Sequence Barrier
+ Wait Strategy
+ Event
+ Event Processor
+ Event Handler
+ Producer



## 场景



## 优劣



## 原理

### 工作原理

结合官方提供的案例（examples/.../longevent/legacy/LongEventMain.java）调试 Disruptor 工作流程。



### 高性能实现原理



## 应用

[User Guide](https://lmax-exchange.github.io/disruptor/user-guide/index.html)

Disruptor 支持3种编码风格（Lambda、Lagacy、MethodRef）。