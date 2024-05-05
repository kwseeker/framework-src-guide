# Redisson分布式锁实现原理

Redisson分布式锁接口实现 `java.util.concurrent.locks.Lock` 接口，意味着 Redisson 的锁接口具有和 JUC 锁接口相同的语义。

Redisson 的锁都实现了**同步**、**异步**、**响应式**接口。

`Redisson`（3.29.0）几种分布式锁：

![](/home/arvin/mywork/framework-src-guide/docs/java/redisson/imgs/redisson-locks-uml.png)

+ RedissonLock
+ RedissonFairLock
+ RedissonReadLock
+ RedissonWriteLock
+ RedissonReadWriteLock
+ RedissonFencedLock
+ RedissonSpinLock
+ RedissonMultiLock
+ RedissonRedLock (已废弃)
+ RedissonSemaphore 
+ RedissonPermitExpirableSemaphore
+ RedissonCountDownLatch



## 各种锁实现原理

### RedissonLock

**类**:`RedissonLock`, 源码单元测试类：`RedissionLockTest`。

+ `RLock`

  继承`JUC` `Lock`接口以及`RLockAsync`拓展的异步接口，拓展了可以自动释放锁的接口。

+ `RExpirable(I)`

  + `RObject`
  + `RObjectAsync`
  + `RExpirableAsync`

  集成`RObject`以及`RExpireableAsync`接口，拓展了生存周期（过期时间）相关的控制接口。

+ `CommandSyncService(C)`

  + `CommandAsyncService(C)`
  + `CommandExecutor (I, 客户端命令执行器)`

+ `LockPubSub`

**加锁流程**：

![](/home/arvin/mywork/framework-src-guide/docs/java/redisson/imgs/Redisson可重入锁工作流程.png)

上图没有解释命令执行流程。

１）通过锁名(`key`名)计算出`Redis`槽点，进而获取所在`Redis`服务实例（这一步对多`Redis`节点才有用，单节点只有一个）。

  ２）创建批处理操作服务(用于将操作同步到从库), 然后创建`CommandBatchService`（示例中创建的是`CommandSyncService`）。

  ​		负责执行`Redis`命令`RedisCommand`(示例中创建的是`RedisStrictCommand`实例)。

  ​		执行器`RedisBatchExecutor`，执行器execute()方法并不负责直接与`redis`服务端联系执行命令。只是给entry(某个`Redis Node`实例)的`LinkedBlockingDeque`中加入待执行命令，就像Java线程池一样，submit等方法只是给线程池任务队列中添加任务。

  ３）通过`CommandBatchService`异步写`EVAL`命令(Future Promise channel写)。

```java
// 加锁逻辑就是函数中这段Lua代码
// 因为Redis lua 脚本执行具有原子性，不需要担心下面使用了三个命令执行加锁操作会被打断
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);
        return evalWriteAsync(getName(), 
                              LongCodec.INSTANCE, 
                              command,
                              // 锁(key=lockName)不存在则添加，Hash key(lockName)  itemKey(threadId)加一并设置锁超时自动释放时间，返回nil
                              "if (redis.call('exists', KEYS[1]) == 0) then " +
                                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                      "return nil; " +
                              "end; " +
                              // 锁已经存在，则判断占有锁的是不是当前线程，是当前线程则重入计数加１，并重新设置超时时间, 返回nil
                              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                     "return nil; " +
                              "end; " +
                              // 锁已存在并且被其他线程占用，则返回剩余超时时间
                              "return redis.call('pttl', KEYS[1]);",
                              // KEYS[1]
                              Collections.singletonList(getName()), 
                              // ARGV[1] 
                              internalLockLeaseTime, 
                              // ARGV[2]
                              getLockName(threadId));
}
```

  ​		从上面可以看到`Redission`实现可重入锁完全是通过`Lua`脚本实现的。用到了`Hash`存储结构(使用`Hash`是为了实现可重入,记录重入次数)，先判断锁是否存在不存在则创建并设置重入计数为１并设置超时时间，已存在则判断是不是当前线程持有锁，若当前线程持有锁则重入计数加１，不是当前线程持有锁则返回剩余超时时间（TTL）。

> 超时时间默认是30s

  ４）如果没有设置TTL时间就直接退出。有的话会同步等待超时之后执行释放。

```java
// 锁释放的逻辑

```

  

### RedissonFairLock

公平锁。



### RedissonReadLock

读锁。



### RedissonWriteLock

写锁。



### RedissonReadWriteLock

读写锁。



### RedissonFencedLock



### RedissonSpinLock

### RedissonMultiLock

联锁。



## RedissonRedLock (已废弃)

红锁。



## RedissonSemaphore 



## RedissonPermitExpirableSemaphore



## RedissonCountDownLatch

