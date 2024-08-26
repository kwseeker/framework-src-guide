# 事务消息原理

事务消息交互流程：

![](https://rocketmq.apache.org/zh/assets/images/transflow-0b07236d124ddb814aeaf5f6b5f3f72c.png)

以一个转账的案例从源码分析下事务消息到底是怎么协调本地事务的？

案例源码参考附录。





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

```
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



