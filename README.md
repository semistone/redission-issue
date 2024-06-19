# redission-issue

#### Test
- run 1000 set/get request parallelly.
- use only one connection

#### Reproduce Step
[source](app/src/main/java/org/example/App.java)

- ./gradlew run
 
it's not always failure, but in my laptop,
about 5 times will fail once.

error logs
```text
 ./gradlew run                                                                                130 ↵ yinchin.chen@JP-MY3R76R99J

> Task :app:run
SLF4J(W): No SLF4J providers were found.
SLF4J(W): Defaulting to no-operation (NOP) logger implementation
SLF4J(W): See https://www.slf4j.org/codes.html#noProviders for further details.
Jun 19, 2024 6:15:34 PM io.netty.resolver.dns.DnsServerAddressStreamProviders <clinit>
WARNING: Can not find io.netty.resolver.dns.macos.MacOSDnsServerAddressStreamProvider in the classpath, fallback to system defaults. This may result in incorrect DNS resolutions on MacOS. Check whether you have a dependency on 'io.netty:netty-resolver-dns-native-macos'
ping success
org.redisson.client.RedisTimeoutException: Unable to acquire connection! java.util.concurrent.CompletableFuture@3c838224[Completed exceptionally: java.util.concurrent.CancellationException]Increase connection pool size. Node source: NodeSource [slot=0, addr=null, redisClient=null, redirect=null, entry=null], command: (SET), params: [test986, PooledUnsafeDirectByteBuf(ridx: 0, widx: 7, cap: 256)] after 3 retry attempts
        at org.redisson.command.RedisExecutor$1.run(RedisExecutor.java:278)
        at io.netty.util.HashedWheelTimer$HashedWheelTimeout.run(HashedWheelTimer.java:706)
        at io.netty.util.concurrent.ImmediateExecutor.execute(ImmediateExecutor.java:34)
        at io.netty.util.HashedWheelTimer$HashedWheelTimeout.expire(HashedWheelTimer.java:694)
        at io.netty.util.HashedWheelTimer$HashedWheelBucket.expireTimeouts(HashedWheelTimer.java:781)
        at io.netty.util.HashedWheelTimer$Worker.run(HashedWheelTimer.java:494)
        at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
        at java.base/java.lang.Thread.run(Thread.java:1583)
org.redisson.client.RedisTimeoutException: Unable to acquire connection! java.util.concurrent.CompletableFuture@3c838224[Completed exceptionally: java.util.concurrent.CancellationException]Increase connection pool size. Node source: NodeSource [slot=0, addr=null, redisClient=null, redirect=null, entry=null], command: (SET), params: [test986, PooledUnsafeDirectByteBuf(ridx: 0, widx: 7, cap: 256)] after 3 retry attempts
        at org.redisson.command.RedisExecutor$1.run(RedisExecutor.java:278)
        at io.netty.util.HashedWheelTimer$HashedWheelTimeout.run(HashedWheelTimer.java:706)
        at io.netty.util.concurrent.ImmediateExecutor.execute(ImmediateExecutor.java:34)
        at io.netty.util.HashedWheelTimer$HashedWheelTimeout.expire(HashedWheelTimer.java:694)
        at io.netty.util.HashedWheelTimer$HashedWheelBucket.expireTimeouts(HashedWheelTimer.java:781)
        at io.netty.util.HashedWheelTimer$Worker.run(HashedWheelTimer.java:494)
        at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
        at java.base/java.lang.Thread.run(Thread.java:1583)
        Suppressed: java.util.concurrent.CancellationException
                at org.redisson.command.RedisExecutor$1.run(RedisExecutor.java:274)
                ... 7 more
        Suppressed: java.lang.Exception: #block terminated with an error
                at reactor.core.publisher.BlockingSingleSubscriber.blockingGet(BlockingSingleSubscriber.java:104)
                at reactor.core.publisher.Mono.block(Mono.java:1779)
                at org.example.App.main(App.java:52)
==== connection never recover

```

and this connection never recover.


### Tracing bug
- ConnectionsHolder.acquireConnection()
- AsyncSemaphore.acquire()
- and AsyncSemaphore.listeners have many objects.

```java
    public CompletableFuture<T> acquireConnection(RedisCommand<?> command) {
        CompletableFuture<T> result = new CompletableFuture<>();

        CompletableFuture<Void> f = acquireConnection();
        f.thenAccept(r -> {
            connectTo(result, command); <=== never come to here
        });
        result.whenComplete((r, e) -> {
            if (e != null) {
                f.completeExceptionally(e);
            }
        });
        return result;
    }
```
and I am sure the connection still OK. because could still see ping success every second.