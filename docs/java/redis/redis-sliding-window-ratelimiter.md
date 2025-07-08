# Redis实现滑动窗口限流

滑动窗口通常包含多个子窗口，窗口滑动是指根据时间不断地插入新的子窗口并删除过期的子窗口。

通过滑动窗口限流就是统计当前时间窗中所有子窗口请求总数是否超过了限制。 

可以使用 ZSet 实现滑动窗口，以请求时间戳作为分值，每次请求就插入一个元素，统计时先根据当前时间戳计算事件窗开始时间戳，然后 ZREMRANGEBYSCORE 删除过期的数据，最后 ZCARD 读取这段时间所有元素个数。

并发情况下有可能删除掉别的任务还在统计的数据，所以**应该使用 Lua 脚本、事务或者高性能分布式锁保证原子性执行**。

源码实现：

TODO: 实现比较简单，待补充。

Lua脚本：

```java
String luaScript = "local window start = ARGV[1] - 60000\n" 
    + "redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', window start)\n" 
    + "local current requests = redis.call('ZCARD', KEYS[1])\n" 
    + "if current requests < tonumber(ARGV[2]) then\n" 
    + " redis.call('ZADD', KEYS[1], ARGV[1], ARGV[1])\n" 
    + " return 1\n" 
    + "else\n" 
    + " return 0\n" 
    + "end
```

