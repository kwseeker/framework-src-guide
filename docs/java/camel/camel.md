# Camel

Apache Camel 是一个基于EIP的开源框架；它是一个面向消息的中间件，提供基于规则的路由和中介引擎。 

EIP定义了一些不同应用系统之间的消息传输模型，包括常见的Point-to-Point，Pub/Sub模型。

Camel 像是不同的**应用程序之间**的**胶水**。

解决的问题：

主要解决企业应用集成（EAI）中各种异构系统、应用和数据源之间数据共享和交换（主要就是数据转换和传输）。

**使用场景**：

+ 消息队列和消息总线

  Camel 可以与各种消息队列和消息总线集成，如 ActiveMQ、RabbitMQ、Kafka 等。通过 Camel，可以轻松地将消息从一个队列或主题路由到另一个队列或主题，并进行转换、过滤、聚合等操作。

+ 消息汇聚

  比如你有来自不同服务器的消息，有ActiveMQ，RabbitMQ，WebService等，你想把它们都存储到日志文件中。

  这点貌似可以替换 logstash 等工具。

+ 消息分发

  分为两种，顺序分发和并行分发。

  顺序分发时，消息会先到到第一个endpoing，第一个endpoint处理完成后，再分发到下下个endpoint。如果第一个endpoing处理出现故障，那么消息不会被传到第二个endpoint。

  并行分发是将得到的消息同时发送到不同的endpoint，没有先后顺序之分，各个endpoint处理消息也是独立的。

+ 消息转换与集成

  Camel 可以将不同的数据格式进行转换，如 XML、JSON、CSV 等。同时，它也可以与各种数据源集成，如数据库、文件系统、FTP 等。通过 Camel，可以轻松地将数据从一个源转换为另一个源，并进行各种操作。

+ 文件处理和 FTP

  Camel 可以与各种文件系统和 FTP 服务器集成，如本地文件系统、SFTP、FTP 等。通过 Camel，可以轻松地将文件从一个位置复制到另一个位置，并进行转换、过滤、聚合等操作。

+ 规则引擎

  可以使用 Spring Xml, Groovy 这类DSL来定义route，这样无需修改代码，就能达到修改业务逻辑的目的。

**类似技术**：

+ Spring Integration
+ Mule ESB



## 案例

### 实现微服务之间的通信



## 参考

+ [Apache Camel 教程](https://www.w3schools.cn/apache_camel/index.html)

+ [Apache Camel 教程](https://www.cnblogs.com/d1012181765/p/15338105.html)

+ [Enterprise Integration Patterns (EIP)](https://www.enterpriseintegrationpatterns.com/)