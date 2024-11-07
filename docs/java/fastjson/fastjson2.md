# Fastjson2

## 高性能原理

Fastjson2 号称是 Java 序列化框架中性能最高的。

**从源码中直观可见的优化手段**：

+ 使用生成器模式（ASM）访问对象属性，避免使用反射

+ 使用缓存机制存储生成的 ObjectWriter 便于复用

+ 细节优化

  + 使用 UNSAFE 写存储序列化后字符的 char 数组，而不是通过数组索引的方式

    > TODO 测试下这种改造到底能提升多少性能？

**另外搜索到优化手段**：

> 待验证。

+ 并行解析
+ 高效的数据结构
+ 假定有序快速匹配算法

## Fastjson的问题

参考：[经过多方调研，最终还是决定禁用FastJson(https://cloud.tencent.com/developer/article/1795697)

## TODO

老版本中 ObjectWriterCreator 实现中 ObjectWriterCreatorLambda （基于 LambdaMetaFactory 反射实现） 和 ObjectWriterCreatorDynamicCompile （基于 JavaCompiler 动态编译实现） 实现原理？

