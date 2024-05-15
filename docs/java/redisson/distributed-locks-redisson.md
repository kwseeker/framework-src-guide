# Redisson分布式锁实现原理

Redisson分布式锁接口实现 `java.util.concurrent.locks.Lock` 接口，意味着 Redisson 的锁接口具有和 JUC 锁接口相同的语义。

Redisson 的锁都实现了**同步**、**异步**、**响应式**接口。

`Redisson`（3.29.0）几种分布式锁：

![](/home/arvin/mywork/framework-src-guide/docs/java/redisson/imgs/redisson-locks-uml.png)

> Lock 接口是JUC中的接口。

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

基本的可重入分布式锁实现。

实现原理：

加锁是借助Hash数据类型通过发送lua脚本执行 hincrby pexpire 实现，同时只有一个线程加锁成功（成功返回nil），未加锁成功的线程先订阅主题  `redisson_lock__channel:{<lockName>}`，再尝试获取信号量（RedissonLockEntry.latch，初始为0）只有成功获取信号量的才可以继续尝试获取锁，获取锁的线程执行完同步代码释放锁会发布释放锁的消息进而给latch++, 这样就可以推动其他等待的线程继续尝试获取锁，从而实现线程同步。



### RedissonFairLock

公平锁。



### RedissonReadLock

分布式读锁，实现了RLock接口，依赖 RedissonLock。



### RedissonWriteLock

分布式写锁，实现了RLock接口，依赖 RedissonLock。



### RedissonReadWriteLock

可重入分布式读写锁实现，依赖 RedissonReadLock、RedissonWriteLock，而这两个类又依赖 RedissonLock。

实现了 JDK ReadWriteLock 接口，依赖 RedissonReadLock 和 RedissonWriteLock，分布式读锁和分布式写锁都实现了RLock接口。

Redisson读写锁实现很简单，基本完全依赖 RedissonLock，仅仅重写了少量方法。

**读读不互斥，读写互斥、写写互斥怎么实现的？**

看源码可以发现区别就是在被重写的方法的 Lua 脚本中。
读写锁同样使用 **Hash** 数据类型实现，KEY是锁名，VALUE是2-3组KV；
读锁额外需要 **String** 数据类型记录线程自己加的哪个重入计数的读锁, 以便解读锁时使用。

第1组KV记录**锁的模式**（只有读锁时是 read 模式、有写锁时变为 write 模式）:

> hset lock-rw mode read

**读锁实现：**

读锁之间不互斥，有线程加了读锁，且只有读锁的情况下，其他线程也可以继续加读锁，只需要增加重入计数

> hset lock-rw <ServiceManager ID>:<threadId> <reentantCount>

但是解读锁时只能解自己加的读锁，就需要额外记录下自己加的哪个重入计数的读锁（下面的 reentrantCount）

>  set {lock-rw}:<ServiceManager ID>:<threadId>:rwlock_timeout:<reentrantCount> 1

**写锁实现：**

已经存在任意读锁或写锁，且加写锁的线程不是当前线程都不能加写锁

> hset lock-rw <ServiceManager ID>:<threadId>:write <reentantCount>



### RedissonFencedLock



### RedissonSpinLock

### RedissonMultiLock

联锁。



## RedissonRedLock (已废弃)

红锁。



## RedissonSemaphore 



## RedissonPermitExpirableSemaphore



## RedissonCountDownLatch

