# 分布式锁设计



## 场景引入

分布式服务中不同线程对同一数据进行写操作都会涉及到分布式锁控制的问题，使用分布式锁时常常会面临以下问题需要解决：

+ 怎么避免多个线程同时修改同一个数据？（互斥性）
+ 如果某线程获取到锁之后执行比较耗时的操作，其他线程迟迟拿不到锁其他线程怎么办？（设置超时时间取消）
+ 如果某线程获取锁之后出异常没有释放锁怎么办？（设置锁超时自动释放）
+ 如果某个线程获取锁之后工作还没做完就超时被释放了怎么办？（需要做异步续命）
+ 如果在给某个复合操作加分布式锁时发现里面的子操作需要获取同一个分布式锁做同步控制怎么办？（添加可重入支持）
+ 万一分布式锁底层的服务挂掉怎么办？（对底层服务做高可用）

应用场景：

+ 秒杀（抢购类场景）

+ 多端写

+ 接口幂等校验

  比如两个人同时为一个账单付费，其中一个支付成功，返回“支付成功”，另一个人就不能支付，但是也要返回“支付成功”。

  

## 分布式锁设计要点

锁功能要求，不仅仅适用于分布式锁。

+ **互斥性**

+ **请求锁超时取消**

  请求锁一直获取不到，要支持超时取消请求。

  借鉴`FutureTask`实现等待获取锁超时退出。并不适合直接使用`FutureTask`。

+ **锁超时自动释放 & 异步续命**

  `redis` 的`setex`（相当于`set`和`expire`的原子组合）指令可以设置超时时间；
  使用锁超时虽然可以避免死锁，但也可能带来不确定性问题（比如某个线程还没来得及做完需同步的操作，锁就释放掉了，其他分布式节点上的线程取锁成功，导致出现竞争），需要做“异步续命”。
  
+ **阻塞和非阻塞**

  未抢到锁直接返回下单失败，以及长时间下单不成功都会给用户带来很不好的体验。
  
+ **可重入性**

  可以参考`ReentrantLock`实现可重入性。

+ **高可用**

  `Redis`通常会部署成集群, `Redis`集群是AP架构，不能确保数据强一致性。

  所以`Redisson`实现分布式锁还是有可能因为`Redis`集群节点同步锁数据的时候因为数据不一致导致出现问题。

  基于`Zookeeper`实现分布式锁相对`Redis`更可靠些，但是性能比`Redis`差很多。




## 分布式锁常见问题

+ **锁失效**

  `Redisson`锁失效：  
  
  1）锁超时失效：使用异步续命解决； 
  2）`Redis`集群一致性问题导致的锁失效：使用`Redisson`红锁解决（会牺牲性能）。
  
  `Redisson`红锁加锁会在所有`Redis`节点上请求锁，当有超过一半节点加锁成功才会认为加锁成功。
  这种操作类似`ZK`（所有节点锁数据同步成功之后才返回加锁结果）。
  
+ **如何提高锁的性能**

  技巧（只能适用于部分业务）：
  
  比如100万用户抢1000件商品，可以将这1000件商品拆分成10份，用10个分布式锁管理每一份商品的抢购。
  虽然有失去公平性但是性能可以提升10倍。
  
+ **如何使用数据库实现分布式锁**

  1）使用select ... for update加行锁;  
  
  2）使用数据版本号+`CAS`的思想。
  
+ **如何实现幂等性检查**

  重试与幂等性问题。
  
  比如：两个重复请求，第一次请求成功插入数据，第二次再次请求插入（通常数据库会返回失败，
  但是以人的角度看应该是返回正确，而且不应该重复执行）  
  
  实际场景：  
  
  + 多次删除购物车同一商品（可能打开两个页面，每个页面做一次删除操作），期望是后台不会重复删除，且不会报错。
  
    解决方案（三个步骤）：  
  
    1）锁  
  
    为每个请求设置一个唯一编号（重复请求拿这个编号识别是否为同一个请求），将此编号作为`Redisson`锁的key；请求进来先加锁，避免重复请求重复执行；
    执行完毕后，将编号以及操作结果存储到缓存中。  
  
    唯一编号可以通过`UUID`、`MD5`等方式实现。
  
    怎么定义是不是同一个请求？
  
    2）判  
  
    重复请求进来后获取锁后，先判断缓存中有没有此编号的记录，有的话直接获取上次执行结果返回。  
  
    3）更新  
  
    更新缓存续命时间。
  

  

## 分布式锁实现方案

### `Redis setex lua` 简单实现

参考`RedisDistributedLockImpl.java`。

> 之所以用`setex`是因为`setnx`没有超时释放，且`setex`是原子命令。

### `Redission`

参考 [distributed-locks-redisson.md](./distributed-locks-redisson.md) 。