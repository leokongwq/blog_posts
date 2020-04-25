---
layout: post
comments: true
title: zookeeper使用之curator
date: 2018-06-17 18:26:10
tags:
- zookeeper
- curator
categories:
- 架构
---

## 前言

Apache [Curator](http://curator.apache.org/)是Netflix开源的操作Zookeeper的，功能非常强大的客户端。提供了很多非常好用的功能来帮助我们构建分布式应用。

## 分布式锁

Curator 提供的分布式锁分为以下三类：

- 可重入公平锁 InterProcessMutex
- 不可重入的非公平锁 InterProcessSemaphoreMutex
- 可重入的读写锁 InterProcessReadWriteLock
- 锁集合 InterProcessMultiLock

<!-- more -->

### InterProcessMutex

Curator提供了`InterProcessMutex`来实现该功能。样例代码如下：

```java InterProcessMutex
//创建锁
InterProcessLock lock = new InterProcessMutex(client, lockPath);
//获取
lock.acquire(1，TimeUnit.SECONDS)
try {
//业务代码
} finally {
    lock.release();
}
```

实现原理分析：

`InterProcessMutex`通过创建一个 Zookeeper 临时，顺序结点来实现锁的公平获取，代码如下：

```java
 @Override
public void acquire() throws Exception {
   if ( !internalLock(-1, null) ) {
       throw new IOException("Lost connection while trying to acquire lock: " + basePath);
   }
}

private boolean internalLock(long time, TimeUnit unit) throws Exception {
   /*
      Note on concurrency: a given lockData instance
      can be only acted on by a single thread so locking isn't necessary
   */
   
   //下面的代码实现：可重入性。
   Thread currentThread = Thread.currentThread();
   LockData lockData = threadData.get(currentThread);
   if ( lockData != null )
   {
       // re-entering 
       lockData.lockCount.incrementAndGet();
       return true;
   }
   //创建：临时，顺序结点 
   String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
   if ( lockPath != null )
   {
       LockData newLockData = new LockData(currentThread, lockPath);
       threadData.put(currentThread, newLockData);
       return true;
   }

   return false;
}
```

创建：临时，顺序结点 

```java LockInternals.attemptLock
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception {
while ( !isDone )
   {
       isDone = true;

       try
       {   
           // 通过LockInternalsDriver创建Zookeeper结点
           ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
           // 判断是否获取了锁， 其实就是判定该线程创建的临时结点的顺序是否最小
           hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
       }
       catch ( KeeperException.NoNodeException e )
       {
           // gets thrown by StandardLockInternalsDriver when it can't find the lock node
           // this can happen when the session expires, etc. So, if the retry allows, just try it all again
           if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
           {
               isDone = false;
           }
           else
           {
               throw e;
           }
       }
   }
}   
```

```java
 private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
   boolean     haveTheLock = false;
   boolean     doDelete = false;
   try
   {
       if ( revocable.get() != null )
       {
           client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
       }
       // 循环来获取锁
       while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
       {
           //获取所有的排序后的子节点列表（升序排列） 
           List<String>        children = getSortedChildren();
           String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash
           // 判断该进程或线程创建的子节点：sequenceNodeName 是否是第一个
           PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
           if ( predicateResults.getsTheLock() )
           {
               haveTheLock = true;
           }
           else
           {
               String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();
                // JVM 进程能同步，
               synchronized(this)
               {
                   try 
                   {
                       // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                       client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                       if ( millisToWait != null )
                       {
                           millisToWait -= (System.currentTimeMillis() - startMillis);
                           startMillis = System.currentTimeMillis();
                           if ( millisToWait <= 0 )
                           {
                               doDelete = true;    // timed out - delete our node
                               break;
                           }
                            // 线程超时等待
                           wait(millisToWait);
                       }
                       else
                       {
                          // 线程等待(需要其它线程唤醒，其它线程在释放锁时会进行唤醒)
                           wait();
                       }
                       
                        //  watcher 的执行在另一个线程中，会唤醒等待锁的线程
                   }
                   catch ( KeeperException.NoNodeException e ) 
                   {
                       // it has been deleted (i.e. lock released). Try to acquire again
                   }
               }
           }
       }
   }
   catch ( Exception e )
   {   
        // 处理线程中断  
       ThreadUtils.checkInterrupted(e);
       doDelete = true;
       throw e;
   }
   finally
   {
        //被中断后，删除结点，释放锁（一点会删除成功，具体实现查询源代码）
       if ( doDelete )
       {
           deleteOurPath(ourPath);
       }
   }
   return haveTheLock;
}
```

### InterProcessSemaphoreMutex

该功能的锁是由：`InterProcessSemaphoreMutex`实现的，具体原理就不分析代码。

提一句：该锁的功能是通过：`InterProcessMutex`和`InterProcessSemaphoreV2`实现的。

`InterProcessSemaphoreV2`：从名称上来说，该类是一个信号量，也就是说它可以提供N个可用的许可，可以用来管理同时访问共享资源的并发数。

> 许可数量为:1的信号量可以当做锁来用。

### InterProcessReadWriteLock

读写锁和JDK提供的：`ReadWriteLock`功能是类似的

```java
 @Test
public void testReadWriteLock() {
   String lockPath = "/config/lock/123";
   InterProcessReadWriteLock readWriteLock = new InterProcessReadWriteLock(client, lockPath);

   InterProcessMutex readLock = readWriteLock.readLock();
   InterProcessMutex writeLock = readWriteLock.writeLock();

}
```

特点总结：

1. 可重入，读锁和写锁都是可重入的。
2. 读锁非互斥，写锁是互斥的，只能有一个客户端获取
3. 读写是互斥的
4. 支持锁降级。支持写锁降级为读锁，读锁不能升级为写锁。


### InterProcessMultiLock

```java
InterProcessMultiLock lock = new InterProcessMultiLock(List<InterProcessLock> locks);

或

InterProcessMultiLock lock = new InterProcessMultiLock(CuratorFramework client,
                             List<String> paths);
```

`InterProcessMultiLock`维护一组锁。在获取锁时，只有获取所有的锁时才返回，释放时会释放所有的锁。

## Leader选举

在分布式系统中，Leader选举是一个非常基础和重要的能力。通过Zookeeper和Curator能非常容易的实现Leader选举。

Curator提供了两种机制来实现Leader选举：

### 方法一：LeaderSelector

直接上代码：

```java
private String leaderSelectionPath = "/config/vip/db/master";

@Test
public void testLeaderElection() throws Exception {
   ExecutorService threadPool = Executors.newFixedThreadPool(5);

   for (int i = 0; i < 5; i++) {
       threadPool.execute(new Contender(buildLeaderSelectorListener()));
   }

   Thread.sleep(1000 * 1000);

   threadPool.shutdown();
}
/**
* 公平的选举（顺序性）
*/
class Contender implements Runnable {
   private LeaderSelectorListener listener;

   Contender(LeaderSelectorListener listener) {
       this.listener = listener;
   }

   @Override
   public void run() {
       LeaderSelector selector = new LeaderSelector(client, leaderSelectionPath, listener);
       // 自动加入Leader选举
       selector.autoRequeue();
       //开始进行选举，不过该方法会立即返回。选举的结果是通过异步回调实现的
       selector.start();
   }
}
private LeaderSelectorListener buildLeaderSelectorListener() {
   LeaderSelectorListener listener = new LeaderSelectorListenerAdapter() {
        //该方法只有在该客户端被选为Leader才会被调用。
        //方法返回就表示客户端放弃Leader角色。
       @Override
       public void takeLeadership(CuratorFramework client) throws Exception {
          // 模式业务操作
            System.out.println("Current Leader is : " + Thread.currentThread().getName());
           Thread.sleep(5000)
       }
   };
   return listener;
}
```

### 方法二：LeaderLatch

```java
 @Test
public void testLeaderRandomElection() throws Exception {
   ExecutorService threadPool = Executors.newFixedThreadPool(5);

   for (int i = 0; i < 5; i++) {
       LeaderLatch leaderLatch = new LeaderLatch(client, leaderSelectionPath, String.valueOf(i));
       threadPool.execute(new RandomSelectLeader(leaderLatch));
   }

   Thread.sleep(1000 * 1000);

   threadPool.shutdown();
}

class RandomSelectLeader implements Runnable {

   private LeaderLatch leaderLatch;

   RandomSelectLeader(LeaderLatch leaderLatch) {
       this.leaderLatch = leaderLatch;
   }

   @Override
   public void run() {
       try {
           //强烈建议添加Listener来监听链接变化，　处理Leader丢失问题
           leaderLatch.addListener(new LeaderLatchListener() {
               @Override
               public void isLeader() {
                   System.out.println("I'am Selected to be a Leader" + Thread.currentThread().getName());
               }

               @Override
               public void notLeader() {
                   System.out.println("I'am Not Leader" + Thread.currentThread().getName());
               }
           });
           //调用该方法后才能开始参与选举（随机的）
           leaderLatch.start();
           //死等，　直到被选为Leader
           leaderLatch.await();

           System.out.println("Current Leader is : " + Thread.currentThread().getName());

           Thread.sleep(2000);
       } catch (Exception e) {
           e.printStackTrace();
       } finally {
           try {
               // 退出选举，如果自己是Leader，则放弃Leader位置，其它的成员就可以再次选举
               leaderLatch.close();
           } catch (Exception e) {
               e.printStackTrace();
           }
       }
   }
}
```

#### 小结

推荐使用`LeaderSelector`来实现Leader选举，因为参加选举的结点代码编写简单，能自动完成丢失Leader角色后从新参加选举的功能，而且能灵活的控制释放Leader角色的时机。

## 分布式栅栏

### DistributedBarrier

```java
@Test
public void testDistributedBarrier() throws Exception {
   String barrierPath = "/config/barriers";
   DistributedBarrier barrier = new DistributedBarrier(client, barrierPath);

   ExecutorService executorService = Executors.newCachedThreadPool();

   for (int i = 0; i < 5; i++) {
       executorService.execute(new Worker(barrier));
   }

   Thread.sleep(1000 * 5);

   //主线程设置结点，worker线程等待
   barrier.setBarrier();

   Thread.sleep(1000 * 10);
   System.out.println("ALL Done!");

   barrier.removeBarrier();
}

class Worker implements Runnable {

   private DistributedBarrier barrier;

   Worker(DistributedBarrier barrier) {
       this.barrier = barrier;
   }

   @Override
   public void run() {
       try {
           barrier.waitOnBarrier();
           System.out.println(Thread.currentThread().getName() + " >>>> Finished");
       } catch (Exception e) {
           e.printStackTrace();
       }
   }
}
```

### DistributedDoubleBarrier

双栅栏允许客户端在计算的开始和结束时同步。当足够的进程加入到双栅栏时，进程开始计算， 当计算完成时，离开栅栏。

构造函数为：

```java
public DistributedDoubleBarrier(CuratorFramework client,
                                String barrierPath,
                                int memberQty)
Creates the barrier abstraction. memberQty is the number of members in the barrier. When enter() is called, it blocks until
all members have entered. When leave() is called, it blocks until all members have left.
Parameters:
client - the client
barrierPath - path to use
memberQty - the number of members in the barrier
```

`memberQty`是成员数量，当`enter`方法被调用时，成员被阻塞，直到所有的成员都调用了`enter`。 当`leave`方法被调用时，它也阻塞调用线程， 直到所有的成员都调用了`leave`。

`DistributedDoubleBarrier`会监控连接状态，当连接断掉时`enter`和`leave`方法会抛出异常。

下面是一个例子：

```java
private static final int QTY = 5;

@Test
public void testDistributedDoubleBarrier() throws Exception {
   String barrierPath = "/config/barriers";

   ExecutorService executorService = Executors.newFixedThreadPool(QTY);
   for (int i = 0; i < QTY; i++) {
       final DistributedDoubleBarrier barrier = new DistributedDoubleBarrier(client, barrierPath, QTY);
       executorService.execute(new Worker(barrier));
   }

   Thread.sleep(1000 * 5);
   System.out.println("ALL Done!");

   executorService.shutdown();
   executorService.awaitTermination(10, TimeUnit.MINUTES);
}

class Worker implements Runnable {

   DistributedDoubleBarrier doubleBarrier;

   Worker(DistributedDoubleBarrier doubleBarrier) {
       this.doubleBarrier = doubleBarrier;
   }

   @Override
   public void run() {
       try {
           //等待所有客户端都到达
           doubleBarrier.enter();
           System.out.println("I'am arrival");
           System.out.println(Thread.currentThread().getName() + " >>>> Finished");
           //等待所有客户端都到达
           doubleBarrier.leave();
           System.out.println("I'am leave");
       } catch (Exception e) {
           e.printStackTrace();
       }
   }
}
```

## 分布式计数器

### SharedCount

`SharedCount` 管理一个共享的整型数字。所有监听该同一个Zookeeper Path的客户端都能获取到该整型数字的最新值。

例子：

```java
public class SharedCountTest extends BaseTest {

    private static final String COUNTER_PATH = "/config/conunter/total";

    @Test
    public void testSharedCount() throws Exception {
        SharedCount sharedCount = new SharedCount(client, COUNTER_PATH, 0);
        sharedCount.start();

        ExecutorService executorService = Executors.newFixedThreadPool(3);

        for (int i = 0; i < 3; i++) {
            final SharedCount count = new SharedCount(client, COUNTER_PATH, 0);
            count.start();
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    count.addListener(new SharedCountListener() {
                        @Override
                        public void countHasChanged(SharedCountReader sharedCount, int newCount) throws Exception {
                            System.out.println(Thread.currentThread().getName() + "====" + newCount);
                        }

                        @Override
                        public void stateChanged(CuratorFramework client, ConnectionState newState) {
                            System.out.println(newState);
                        }
                    });

                    try {
                        Thread.sleep(1000 * 5);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        Thread.sleep(1000 * 3);
        sharedCount.setCount(123);
        Thread.sleep(1000 * 3);
        sharedCount.setCount(456);
        Thread.sleep(1000 * 20);
        sharedCount.close();
    }
}
```

> 注意： 如果连续两次调用`setCount`方法，在客户端只能观察到最后一次的结果。`trySetCount` 只有当该客户端的缓存的值和服务端保存的值一致才能设置成功，否则该客户端的值会自动更新（`trySetCount`返回false）。

### DistributedAtomicLong

`DistributedAtomicLong` 是一个原子更新的计数器。它在更新值时，第一次尝试采用乐观锁机制，如果更新失败，则使用`InterProcessMutex`来实现更新。 不管采用哪种机制，它都会采用重试机制，直到更新成功。

一个例子：

```java
public class TestDistributedAtomicLong extends BaseTest {

    private static final String COUNTER_PATH = "/config/conunter/123";

    @Test
    public void testDistributedAtomicLong() throws Exception {
        RetryPolicy retryPolicy = new RetryForever(10);
        DistributedAtomicLong atomicLong = new DistributedAtomicLong(client, COUNTER_PATH, retryPolicy);

        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            final RetryPolicy retryPolicyTmp = new RetryForever(10);
            final DistributedAtomicLong distCounter = new DistributedAtomicLong(client, COUNTER_PATH, retryPolicyTmp);

            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println(distCounter);

                    AtomicValue<Long> result = null;
                    try {
                        result = distCounter.increment();
                        if (result.succeeded()) {
                            System.out.println(Thread.currentThread().getName() + "===> current value = " + result.preValue());
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });
        }

        Thread.sleep(1000 * 5);

        System.out.println("After all current value is = " + atomicLong.get().postValue());
    }
}
```

> 注意：在使用`DistributedAtomicLong`时，你必须首先检查`AtomicValue.succeeded()`的返回值。如果操作成功则该方法的返回`true`，否则返回`false`，表示原子更新失败。如果更新成功，则可以通过`preValue`和`postValue`获取更新前后的值。

## 缓存

### PathChildrenCache

`PathChildrenCache` 用来观察一个`ZNode`。无论该结点下新增子节点，删除子节点还是子节点更新，路径缓存都会更新它的状态来包含当前的结点集合（包括状态和数据）。

一个例子：

```java
public class PathChildrenCacheTest extends BaseTest {

    private static final String CACHE_PATH = "/config/cache";
    
    @Test
    public void testPathChildrenCache() throws Exception {
        PathChildrenCache pathChildrenCache = new PathChildrenCache(client, CACHE_PATH, true);
        pathChildrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);

        pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
                System.out.println(event);
            }
        });
        Thread.sleep(1000 * 50);

        List<ChildData> childDataList = pathChildrenCache.getCurrentData();
        for (ChildData childData : childDataList) {
            System.out.println(childData);
        }

        pathChildrenCache.close();
    }
}
```

### NodeCache

`NodeCache`用来观察一个`ZNode`。当该结点的数据被更新，或被删除，`NodeCache`会同步到结点当前最新的数据，如果结点被删除，则`NodeCache`包含的数据为null。

一个例子：

```java
public class NodeCacheTest extends BaseTest {

    private static final String CACHE_NODE_PATH = "/config/cache/123";

    @Test
    public void testNodeCache() throws Exception {
        final NodeCache nodeCache = new NodeCache(client, CACHE_NODE_PATH);
        nodeCache.start(true);
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                System.out.println(new String(nodeCache.getCurrentData().getData()));
            }
        });

        Thread.sleep(1000 * 50);
        nodeCache.close();
    }
}
```

### TreeCache

`TreeCache`是一个工具类，它试图将服务端某个`Path`下所有所有的结点数据缓存到本地。该类会监听指定的ZK路径，处理`update`,`create`,`delete`事件并拉取服务端的数据。你可以通过注册一个Listener来获取数据的变化通知。

一个例子：

```java
public class TreeCacheTest extends BaseTest {

    private static final String CACHE_PATH = "/config/cache";

    @Test
    public void testTreeCache() throws Exception {
        TreeCache treeCache = new TreeCache(client, CACHE_PATH);
        treeCache.getListenable().addListener(new TreeCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
                System.out.println(event);
            }
        });
        treeCache.start();

        Thread.sleep(1000 * 50);

        Map<String, ChildData> dataMap = treeCache.getCurrentChildren(CACHE_PATH);
        for (Map.Entry<String, ChildData> entry : dataMap.entrySet()) {
            System.out.println(entry.getKey() + " =====》" +  new String(entry.getValue().getData()));
        }
        treeCache.close();
    }
}
```

## Nodes

### Persistent Node

持久化结点是数据保存在Zookeeper服务端，并且在连接断开，session过期任然存在的结点。

```java
public class PersistentNodeTest extends BaseTest {

    private static final String NODE_PATH = "/persons/sky";

    @Test
    public void testPersistentNode() throws Exception {
        PersistentNode persistentNode = new PersistentNode(
            client,
            CreateMode.PERSISTENT,
            true,
            NODE_PATH,
            "123".getBytes()
        );
        //必须先调用该方法(该方法会创建结点)
        persistentNode.start();

        System.out.println(persistentNode.getActualPath());
        System.out.println(new String(persistentNode.getData()));
        //会删除该结点
//        persistentNode.close();
    }
}
```

### Persistent TTL Node

` `在你需要创建`TTL`节点，但不想通过定期设置数据手动保持它的活动状态时非常有用。`PersistentTtlNode`可以为你完成此操作。 此外，保持活动的方式不会在父节点上生成监视触发器。 它还提供了类似`PersistentNode`的保证，即使通过连接和会话中断，节点也会尝试保持在ZooKeeper中。



### Group Member

## 分布式队列









、、、、

