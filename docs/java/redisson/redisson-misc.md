# Redisson Misc

讲解  org.redisson.misc 下的工具类。

+ **AsyncSemaphore**



## AsyncSemaphore

基于 **CompletableFuture** 实现的**异步非阻塞**信号量。

使用 AsyncSemaphore 首先是获取信号量；然后通过 CompletableFuture thenAccept() 等方法添加业务处理逻辑；业务逻辑执行完成后进行信号量释放；
AsyncSemaphore 借助 AtomicInteger 控制信号量令牌个数；借助 ConcurrentLinkedQueue （队列是FIFO）实现任务排序；
注意未获取到令牌的任务是直接返回了 CompletableFuture 实例，并没有使用while循环等待可用令牌；而是在一个获取令牌的任务执行完毕后释放令牌后，又执行了一次 tryRun() 来推动下一个等待中的任务的执行。

源码很简单：

```java
public class AsyncSemaphore {
	//可用信号量令牌个数
    private final AtomicInteger counter;
    //任务排队，先进先出
    private final Queue<CompletableFuture<Void>> listeners = new ConcurrentLinkedQueue<>();

    public AsyncSemaphore(int permits) {
        counter = new AtomicInteger(permits);
    }
    
    public int queueSize() {
        return listeners.size();
    }
    
    public void removeListeners() {
        listeners.clear();
    }

    //获取信号量，不管有没有获取到，都会返回 CompletableFuture
    //只不过获取到信号量，会返回完成的 CompletableFuture（即result有值，空值NIL也是有值）; 
    //未获取到信号量，会返回未完成的 CompletableFuture（即result无值，null）
    public CompletableFuture<Void> acquire() {
        CompletableFuture<Void> future = new CompletableFuture<>();
        listeners.add(future);
        tryRun();
        return future;
    }
	
    //核心方法
    private void tryRun() {
        while (true) {	//只有后面counter.incrementAndGet() > 0 才会继续循环，即说明还有可用的信号量令牌，继续完成队列中的CompletableFuture
            if (counter.decrementAndGet() >= 0) {//成功获取信号量
                CompletableFuture<Void> future = listeners.poll();
                if (future == null) {	//队列中没有等待的任务
                    counter.incrementAndGet();
                    return;
                }

                if (future.complete(null)) {	//完成CompletableFuture, 即会继续回调 thenAccept() 等方法的任务
                    return;
                }
            }

            //获取信号量失败 或 future.complete(null)失败
            if (counter.incrementAndGet() <= 0) {
                return;
            }
        }
    }

    public int getCounter() {
        return counter.get();
    }

    //释放信号量
    public void release() {
        counter.incrementAndGet();
        tryRun();	//再执行一次tryRun(), 触发队列中等待的CompletableFuture的完成
    }

    @Override
    public String toString() {
        return "value:" + counter + ":queue:" + queueSize();
    }   
}
```

测试代码：

```java
public class AsyncSemaphoreTest {

    @Test
    public void testAsyncSemaphore() throws InterruptedException {
        AsyncSemaphore semaphore = new AsyncSemaphore(1);
        Thread thread1 = new Thread(() -> {
            //thenAccept()只有完成前面的任务（acquire()）才会回调
            semaphore.acquire().thenAccept(c -> {
                System.out.println("thread1 acquired semaphore at " + System.currentTimeMillis());
                try {
                    //模拟业务处理耗时
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                semaphore.release();
            });
        });
        Thread thread2 = new Thread(() -> {
            semaphore.acquire().thenAccept(c -> {
                System.out.println("thread2 acquired semaphore at " + System.currentTimeMillis());
                try {
                    //模拟业务处理耗时
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                semaphore.release();
            });
        });
        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();
    }
}
```

