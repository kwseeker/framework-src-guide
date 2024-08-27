# 事务消息原理

**事务消息一些使用场景**：

+ 比如订单上下游业务异步执行（分布式大事务拆分）
+ Redis 缓存和 MySQL 异步同步（为何不用普通消息？因为事务消息有主动检查，多了一层保险）
+ ...



## 事务消息工作流程

**事务消息交互流程**：

![](https://rocketmq.apache.org/zh/assets/images/transflow-0b07236d124ddb814aeaf5f6b5f3f72c.png)

以一个转账的案例从源码分析下事务消息到底是怎么协调本地事务的？

事务消息处理源码流程参考 transaction-message.drawio，看源码和上述流程一致。

案例源码参考附录。

`RocketMQ事务消息`属于几种分布式事务解决方案中的 `可靠消息最终一致性`方案。

貌似也是使用最多的一种方案（2PC/3PC性能低、复杂业务回滚成本太高；TCC实现相对复杂、业务侵入高；Saga不保证隔离性可能因为脏写导致无法补偿）；RocketMQ 事务消息相对实现简单、性能较高，不过**无法确保消费者消费消息时后续业务执行成功**；以转账为例，A 账户扣款成功后，消费者收到转账消息给 B 账户加款可能执行失败，这时 A 账户扣款是无法回滚的，不过可以使用 RocketMQ 重试机制进行**重试**，超过重试限制次数后，需要**记录失败日志并进行告警后续手动处理**。

**RocketMQ事务消息 和 2PC、3PC、TCC 对比最大区别是没有大事务，仅仅保证核心业务和事务消息发送的一致性，下游业务执行结果需要开发者自行保证（失败重试、重试超次数后手动处理），实现简单，性能高**。

因为异常情况都是很稀少的，手动处理也可以忍受。

> 常见分布式事务理论方案：
>
> + 基于 XA 的 2PC、3PC
>
> + 基于业务的 TCC
>
> + 可靠消息最终一致性
>
>   + 本地消息表
>
>     不是分布式事务解决方案，只是为了保证可靠消息最终一致性方案中本地事务和消息发送的一致性（要么都成功，要么都失败）。
>
> + Saga 消息补偿
>
> + 最大努力通知



## 深入分析事务消息设计

为什么事务消息要设计成这种流程？

以官方案例为例：![](https://rocketmq.apache.org/zh/assets/images/tradetrans01-636d42fb6584de6c51692d0889af5c2d.png)

**原始的实现**（5个微服务）：

订单支付成功 -> 发送订单支付成功的广播消息 -> 后面四个服务都订阅此消息主题进行消费

问题分析：

+ **向Broker发送广播消息可能失败，会导致下游业务没有执行**

问题解决：

+ 需要保证核心业务和消息发送的一致性

  可以通过本地消息表确保；而RocketMQ事务消息采用了**半消息机制**：

  先向Broker发送半消息（Broker 将半消息存放在事务半消息主题 **RMQ_SYS_TRANS_HALF_TOPIC**），半消息发送成功再执行本地事务，本地事务执行成功，再向Broker发送COMMIT请求让Broker将消息加入到原始主题，本地事务执行失败就向 Broker 发 Rollback 请求，会将半消息删除（逻辑删除）；对应前面 1-2-3-4 步骤；确保半消息发送失败不会执行本地事务、本地事务执行失败也不会将半消息加入原始主题；

  不过还有问题：**如果本地事务执行成功，但是 COMMIT 请求因为网络原因发送失败呢**？同样会导致下游任务无法执行。

  所以还需要 Rocket 中还在 Broker 中启动了 `TransactionalMessageCheckService` 这个主动检查消息对应本地事务执行状态的非守护线程，定期轮询事务半消息主题中所有队列的半消息，获取对应生产者的连接 Channel，发起 **CHECK_TRANSACTION_STATE** 请求主动触发 Producer 检查本地事务执行状态，然后生产者再向 Broker 发送 **END_TRANSACTION** 请求；对应前面 5-6-7 步骤；

  > 注意这里 CHECK_TRANSACTION_STATE 请求不会等待返回本地事务状态结果，因为这个请求是在两层循环中发起的，为了性能需要尽快返回，对应生产者端收到请求后提交到线程池中就直接返回了。

  

## 案例源码

生产者：

```java
/**
 *  结合一个转账的案例说明 RocketMQ 的事务消息
 *  生产者本地事务执行 A 账户扣款，
 *  消费者执行 B 账户加款
 */
public class TransferTransactionProducer {

    public static final String PRODUCER_GROUP = "transfer_account_pg";
    public static final String DEFAULT_NAMESRVADDR = "127.0.0.1:9876";
    public static final String TOPIC = "TopicTransfer";

    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransferTransactionListenerImpl();

        // 事务消息生产者
        TransactionMQProducer producer = new TransactionMQProducer(PRODUCER_GROUP, Collections.singletonList(TOPIC));
        producer.setNamesrvAddr(DEFAULT_NAMESRVADDR);
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2000), r -> {
                    Thread thread = new Thread(r);
                    thread.setName("client-transaction-msg-check-thread");
                    return thread;
                });
        producer.setExecutorService(executorService);   // 用于检查本地事务执行状态
        producer.setTransactionListener(transactionListener);
        producer.start();

        // 发送事务消息
        try {
            // 消息内容一般是业务对象序列化的字符串的字节数组
            Message msg = new Message(TOPIC,
                    ("transfer bill serialized content ...").getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.sendMessageInTransaction(msg, null);
            System.out.printf("%s%n", sendResult);
        } catch (MQClientException | UnsupportedEncodingException e) {
            e.printStackTrace();
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```

事务监听器：

```java
public class TransferTransactionListenerImpl implements TransactionListener {

    // 本地事务执行状态，一般存储到专门的事务日志服务或者跟随业务服务存储
    private final ConcurrentHashMap<String, LocalTransactionState> transactionStates = new ConcurrentHashMap<>();

    /**
     * 执行本地事务
     */
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        String transactionId = msg.getTransactionId();
        System.out.println("execute local transaction with ID: " + transactionId);
        try {
            // msg 反序列化 ...
            // 执行本地事务，即带着事务处理（比如被 @Transactional 注释）的业务方法
            mockExecLocalTransaction();
            // 执行成功，提交事务
            transactionStates.put(transactionId, LocalTransactionState.COMMIT_MESSAGE);
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            System.out.println("execute local transaction error: " + e);
            // 出现异常回滚
            transactionStates.put(transactionId, LocalTransactionState.ROLLBACK_MESSAGE);
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        LocalTransactionState state = transactionStates.get(msg.getTransactionId());
        if (null != state) {
            return state;
        }
        return LocalTransactionState.UNKNOW;
    }

    private void mockExecLocalTransaction() {
        System.out.println("mock executing local transaction, A count deduct $100");
        //boolean normal = true;
        boolean normal = false;
        // 模拟业务执行中可能出现异常, 本地事务通常会自动回滚
        if (!normal) {
            throw new RuntimeException("some exception occurred");
        }
    }
}
```

消费者：（和普通消费者一样）

```java
public class TransferTransactionConsumer {

    public static final String CONSUMER_GROUP = "transfer_account_cg";
    public static final String DEFAULT_NAMESRVADDR = "127.0.0.1:9876";
    public static final String TOPIC = "TopicTransfer";

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer pushConsumer = new DefaultMQPushConsumer(CONSUMER_GROUP);
        pushConsumer.setNamesrvAddr(DEFAULT_NAMESRVADDR);
        pushConsumer.setConsumeMessageBatchMaxSize(1);
        pushConsumer.subscribe(TOPIC, "*");
        pushConsumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgList, ConsumeConcurrentlyContext context) {
                try {
                    for (MessageExt messageExt : msgList) {
                        System.out.println("mock increase B count balance, messageExt=" + messageExt);
                        // 事务消息消费者端和普通消费者无异，消费消息失败并不会回滚生产者的本地事务，
                        // 比如这个转账的案例场景：A 扣款成功后，消费者端接收到事务消息对 B 进行加款操作，但是由于某种原因失败，并不会回滚 A 扣款操作
                        boolean normal = true;  // true 模拟正常，false 模拟失败
                        mockIncreaseB(normal);
                    }
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                } catch (Exception e) {
                    System.out.printf("consumeMessage error: %s%n", e);
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            }
        });
        pushConsumer.start();
    }

    private static void mockIncreaseB(boolean normal) {
        if (!normal) {
            throw new RuntimeException("mock increase B count balance failed");
        }
    }
}
```



