# Logback

源码主流程还是比较简单的，数据结构也很清晰。

关键是理清楚 Logback 中的各种组件的关系以及是怎么协作的。

参考资料：

+ 《LogBack 0.9.20 中文文档》

  > Zlib 上可以下载，先看源码再看官方文档会理解地很透彻。



## Logback 主流程与组件

- 每个Logger实例都有且仅有一个唯一的名字，根 Logger 的名字是 ROOT

  > 所有 Logger 都可以通过 LoggerContext 查看。

- Logger 之间是一种树型关系，树的根是 ROOT Logger

- 各组件的关系

  **Logger** 通过 **Appender** 输出日志；

  **Appender** 可以绑定多个 **Filter** 对日志事件（LoggingEvent）进行过滤（比如按日志级别选择输出）；

  日志如果有参数输出前需要借助 MessageFormatter 格式化，还需要借助 **Encoder** 进行序列化编码；

  **Encoder** 中通过布局 **Layout** 控制日志序列化的格式 ；

  **Layout** 则是由一组 **Converter** 组成，用于将格式中每一个片段替换成具体的值 ( Logback 支持的所有 Pattern 片段及对应的 Converter 可以在 PatternLayout **DEFAULT_CONVERTER_MAP** 中查看 )。



## 借助 MDC 实现简单的链路追踪

1. 为每个请求生成唯一ID，比如 日期+服务ID+Redis自增ID; 
2. 自定义一个过滤器，拦截请求通过 MDC 类记录请求的唯一 ID （MDC实际是通过 ThreadLocal 传递值）并记录请求深度 Depth。
3. 请求其他基础服务时，将请求ID 和 请求深度 Depth 传递给基础服务请求，基础服务请求深度 +1。
4. 将日志发送给 ELK，后续通过请求ID查询请求的所有日志并通过服务名、Depth 排序，获取请求链路信息。



## 如何自定义布局 Layout

可以参考 Skywalking TraceId  Layout 的实现。

接入 Skywalking 做链路追踪时，往往需要通过日志获取 TraceId， 然后通过 TraceId 去 Skywalking UI 中查询某个请求的链路。

在服务日志中添加 TraceId 需要在 Logback 配置文件中配置 Skywalking 定义的一个 Layout 实现（`org.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout`） ；



