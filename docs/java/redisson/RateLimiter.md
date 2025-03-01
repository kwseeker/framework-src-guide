# Redisson RateLimiter

通过 Lua 脚本实现的。

```java
// 设置限流器的令牌发放速率
public RFuture<Boolean> trySetRateAsync(RateType type, long rate, long rateInterval, RateIntervalUnit unit) {
    return commandExecutor.evalWriteNoRetryAsync(getRawName(), LongCodec.INSTANCE, 	
		RedisCommands.EVAL_BOOLEAN,
        // KEYS[1] 取自 getRawName(), 是限流器的名称
		// ARGV[1] 取自 rate, 比如测试中每2s发5个令牌, 这里是 5
		// ARGV[2] unit.toMillis(rateInterval)， 比如测试中每2s发5个令牌，这里是 2000
        "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);"				// hsetnx myLimiter rate 5
        	+ "redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);"		// hsetnx myLimiter interval 2000
        	+ "return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);", // hsetnx myLimiter type 0
        Collections.singletonList(getRawName()), 	//KEYS 传参
		rate, unit.toMillis(rateInterval), type.ordinal());	//ARGV 传参
}

private <T> RFuture<T> tryAcquireAsync(RedisCommand<T> command, Long value) {
    byte[] random = getServiceManager().generateIdArray();

    return commandExecutor.evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
        // 省略一大段Lua脚本(没有格式化很难看，提出来格式后专门分析)，详细参考后面
		"..."
        // KEYS：
		// 0 = "myLimiter"
        // 1 = "{myLimiter}:value"
        // 2 = "{myLimiter}:value:1432e448-48fc-4355-bb36-c1fbf6508326"
        // 3 = "{myLimiter}:permits"
        // 4 = "{myLimiter}:permits:1432e448-48fc-4355-bb36-c1fbf6508326"
        Arrays.asList(getRawName(), getValueName(), getClientValueName(), getPermitsName(), getClientPermitsName()),
        // ARGV: 1, 1740826873683, random是Redisson ServiceManager生成的ID字节数组
        value, System.currentTimeMillis(), random);
}
```

lua 代码提出来格式化：

```lua
-- hget myLimiter rate
-- hget myLimiter interval
-- hget myLimiter type
local rate = redis.call('hget', KEYS[1], 'rate');
local interval = redis.call('hget', KEYS[1], 'interval');
local type = redis.call('hget', KEYS[1], 'type');
assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

-- 这里只分析 OVERALL 类型实现
-- local valueName = '{myLimiter}:value';		当前可用的令牌数量
-- local permitsName = '{myLimiter}:permits';   发放的令牌
local valueName = KEYS[2];
local permitsName = KEYS[4];
if type == '1' then 
    valueName = KEYS[3];
    permitsName = KEYS[5];
end;
assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); 

-- get {myLimiter}:value 其实是获取可用令牌数量
local currentValue = redis.call('get', valueName); 
local res;
if currentValue ~= false then -- 不为空说明不是第一次请求获取令牌
    	--------------- 这一段是关键实现 begin
    	-- 1 如果有过期的令牌，先删除过期的令牌并恢复可用令牌数量
    	-- zrangebyscore {myLimiter}:permits 0 (1740826873683 - 2000)， 查询过期的令牌
        local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
        local released = 0; 
        for i, v in ipairs(expiredValues) do 
            local random, permits = struct.unpack('Bc0I', v);
            released = released + permits;
        end; 
        -- zrangebyscore {myLimiter}:permits 0 (1740826873683 - 2000)， 删除过期的令牌
        if released > 0 then 
            redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
        	-- 如果当前可用令牌数（currentValue + released）> 最大个数，什么时候会出现这种情况？
            if tonumber(currentValue) + released > tonumber(rate) then 
            	-- 修改可用令牌数
                currentValue = tonumber(rate) - redis.call('zcard', permitsName); 
            else 
            	-- 修改可用令牌数为当前实际可用数量，恢复令牌可用数量
                currentValue = tonumber(currentValue) + released; 
            end; 
        	-- set {myLimiter}:value <currentValue>
            redis.call('set', valueName, currentValue);
        end;
    
		-- 2 先判断可用令牌是否足够，不够就返回回收令牌需要等待的时间ms；足够的话就生成令牌ID存储到{myLimiter}:permits并将可用令牌计数减1
    	-- 如果当前可用的令牌数量不够， ARGV[1]是当前需要的令牌数量，比如 1
        if tonumber(currentValue) < tonumber(ARGV[1]) then 
        	-- zrange {myLimiter}:permits 0 0 withscores, 获取这intervals(测试是2s)内发放的第一个令牌
            local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); 
        	-- ARGV[2] 是当前请求申请令牌时的时间戳，res 其实是需要等待的下次“发放”令牌的时间（回收旧的令牌）
        	-- 可以用于阻塞获取精确控制等待时间， +3 ms 是额外预留一点时间防止还没有过期
            res = 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));
        -- 当前可用令牌足够
    	else 
        	-- zadd {myLimiter}:permits 1 <生成的令牌ID>
            redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1]));  
        	-- decrby {myLimiter}:value 1, 可用令牌计数减1
            redis.call('decrby', valueName, ARGV[1]); 
            res = nil; 
        end; 
    	--------------- end
else -- get {myLimiter}:value 为空的处理，可能是首次获取令牌，也可能限流器过期被自动删除了
        redis.call('set', valueName, rate); 
        redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); 
        redis.call('decrby', valueName, ARGV[1]); 
        res = nil; 
end;

-- 设置限流器过期时间
local ttl = redis.call('pttl', KEYS[1]); 
if ttl > 0 then 
    redis.call('pexpire', valueName, ttl); 
    redis.call('pexpire', permitsName, ttl); 
end; 
return res;
```

