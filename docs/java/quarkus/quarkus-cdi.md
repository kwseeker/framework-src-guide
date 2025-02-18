# Quarkus CDI

主流程参考流程图 [quarkus-core.drawio](./quarkus-core.drawio)。

遗留问题：

+ 拓展组件中的 Bean 类型是如何生成的？

  源码有点复杂没看完，但是从组件 Processor 源码中，经常会看到 `SyntheticBeanBuildItem`，估计这个 BuildItem 的处理就是 Bean 类型生成逻辑的入口。

  比如：

  ```java
  SyntheticBeanBuildItem.ExtendedBeanConfigurator configurator = SyntheticBeanBuildItem
          .configure(RabbitMQClient.class)
          .scope(ApplicationScoped.class)
          .setRuntimeInit()
          .unremovable()
          .supplier(rabbitMQClientSupplier);
  
  if (RabbitMQClients.DEFAULT_CLIENT_NAME.equals(client.getName())) {
      configurator.addQualifier(Default.class);
  } else {
      configurator.addQualifier().annotation(DotNames.NAMED).addValue("value", client.getName()).done();
      configurator.addQualifier().annotation(NamedRabbitMQClient.class).addValue("value", client.getName()).done();
  }
  syntheticBeans.produce(configurator.done());
  ```



