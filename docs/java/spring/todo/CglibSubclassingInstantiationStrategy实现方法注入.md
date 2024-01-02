# CglibSubclassingInstantiationStrategy 实现方法注入原理

+ Spring默认提供的两种实例化策略（InstantiationStrategy）

  + SimpleInstantiationStrategy
  + CglibSubclassingInstantiationStrategy extends SimpleInstantiationStrategy

+ Spring实例化策略的选择

+ Spring方法注入的方式 & CglibSubclassingInstantiationStrategy支持的方法注入

  + Spring方法注入的常用方式 

    + @Autowired setter方法注入
    + @Bean 方法注入
    + @Lookup 方法注入

  + CglibSubclassingInstantiationStrategy支持的方法注入

    + PASSTHROUGH
    + LOOKUP_OVERRIDE
    + METHOD_REPLACER

    > TODO 具体对应的Spring方法注入方式

+ CglibSubclassingInstantiationStrategy 方法注入原理