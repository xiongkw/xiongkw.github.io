---
layout: post
title: Curator-recipes Lock源码分析
categories: [编程, java]
tags: [curator, zookeeper, lock]
---

> jdk的锁作用于同一jvm进程内的不同线程，而这里的`Curator Lock`说的是多个客户端(不同jvm进程)之间的锁

#### 1. 一个例子

```java
public static void main(String[] args) {
    for (int i = 0; i < 10; i++) {
        int finalI = i;
        new Thread(() -> {
            CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
            curator.start();
            InterProcessLock lock = new InterProcessMutex(curator, "/samples/locks");
            try {
                System.out.println("Thread " + finalI+" started");
                lock.acquire(3, TimeUnit.SECONDS);
                System.out.println("Thread " + finalI+" get the lock");
                Thread.sleep(1000);
            } catch (Exception e) {
                e.printStackTrace();
            }finally {
                try {
                    lock.release();
                } catch (Exception e) {
                }
            }
        }).start();
    }
    
    //省略close操作
}
```

#### 2. 源码分析
获取锁
```java
public boolean acquire(long time, TimeUnit unit) throws Exception{
    return internalLock(time, unit);
}
```

internalLock
```java
private boolean internalLock(long time, TimeUnit unit) throws Exception {
    /*
       Note on concurrency: a given lockData instance
       can be only acted on by a single thread so locking isn't necessary
    */

    Thread currentThread = Thread.currentThread();

    LockData lockData = threadData.get(currentThread);
    if ( lockData != null ) {
        // re-entering
        lockData.lockCount.incrementAndGet();
        return true;
    }

    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null ) {
        LockData newLockData = new LockData(currentThread, lockPath);
        threadData.put(currentThread, newLockData);
        return true;
    }

    return false;
}
```

> 使用线程`threadData(ConcurrentHashMap)`保存当前线程锁状态实现锁的可重入，这里是否可以用`ThreadLocal`?

LockInternals.attemptLock
```java
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception {
    final long      startMillis = System.currentTimeMillis();
    final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
    final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
    int             retryCount = 0;

    String          ourPath = null;
    boolean         hasTheLock = false;
    boolean         isDone = false;
    while ( !isDone ) {
        isDone = true;

        try {
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        } catch ( KeeperException.NoNodeException e ) {
            // gets thrown by StandardLockInternalsDriver when it can't find the lock node
            // this can happen when the session expires, etc. So, if the retry allows, just try it all again
            if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) ) {
                isDone = false;
            } else {
                throw e;
            }
        }
    }

    if ( hasTheLock ) {
        return ourPath;
    }

    return null;
}
```

StandardLockInternalsDriver.createsTheLock
```java
public String createsTheLock(CuratorFramework client, String path, byte[] lockNodeBytes) throws Exception {
    String ourPath;
    if ( lockNodeBytes != null ) {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, lockNodeBytes);
    }
    else {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
    }
    return ourPath;
}
```

> 创建锁实际上是在zk指定path创建了一个有序临时节点

```java
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception {
    boolean     haveTheLock = false;
    boolean     doDelete = false;
    try {
        if ( revocable.get() != null ) {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }

        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock ) {
            List<String>        children = getSortedChildren();
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash

            PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if ( predicateResults.getsTheLock() ) {
                haveTheLock = true;
            }
            else {
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                synchronized(this) {
                    try {
                        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null ) {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 ) {
                                doDelete = true;    // timed out - delete our node
                                break;
                            }

                            wait(millisToWait);
                        }
                        else {
                            wait();
                        }
                    }
                    catch ( KeeperException.NoNodeException e ) {
                        // it has been deleted (i.e. lock released). Try to acquire again
                    }
                }
            }
        }
    }
    catch ( Exception e ) {
        ThreadUtils.checkInterrupted(e);
        doDelete = true;
        throw e;
    }
    finally {
        if ( doDelete ) {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```

StandardLockInternalsDriver.getsTheLock
```java
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception {
    int             ourIndex = children.indexOf(sequenceNodeName);
    validateOurIndex(sequenceNodeName, ourIndex);

    boolean         getsTheLock = ourIndex < maxLeases;
    String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

    return new PredicateResults(pathToWatch, getsTheLock);
}
```

> 序号最小的(最先创建有序临时节点)的client将获得锁

竞争失败的client会监听前一个序号节点
```java
String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

synchronized(this) {
    try {
        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
        }
       // ...
```
> 前一个序号节点释放后会通知其去获取锁，这样的好处是每个竞争失败的client只会监听其前一个序号节点，也就是说，当前锁节点释放以后，只会通知到其后序节点的client，而不会通知所有client


锁释放
```java
public void release() throws Exception {
    /*
        Note on concurrency: a given lockData instance
        can be only acted on by a single thread so locking isn't necessary
     */

    Thread currentThread = Thread.currentThread();
    LockData lockData = threadData.get(currentThread);
    if ( lockData == null ) {
        throw new IllegalMonitorStateException("You do not own the lock: " + basePath);
    }

    int newLockCount = lockData.lockCount.decrementAndGet();
    if ( newLockCount > 0 ) {
        return;
    }
    if ( newLockCount < 0 ) {
        throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + basePath);
    }
    try {
        internals.releaseLock(lockData.lockPath);
    } finally {
        threadData.remove(currentThread);
    }
}
```
> 当`newLockCount = 0`时才会释放zk节点，`newLockCount > 0`直接返回(重入锁) 

```java
final void releaseLock(String lockPath) throws Exception {
    client.removeWatchers();
    revocable.set(null);
    deleteOurPath(lockPath);
}
```

> `releaseLock`删除了zk节点

#### 3. 其它锁

* `InterProcessSemaphoreMutex`，不可重入，同一个线程只能锁一次
* `InterProcessReadWriteLock`， 一个读写锁管理一对相关的锁。 一个负责读操作，另外一个负责写操作。 读操作在写锁没被使用时可同时由多个进程使用，而写锁使用时不允许读 (阻塞)。 此锁是可重入的。一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁。
* `InterProcessSemaphore`，一个计数的信号量类似JDK的`Semaphore`
* `InterProcessMultiLock`，组锁，其构造函数的参数为一组锁集合`InterProcessMultiLock(List<InterProcessLock> locks)`，`acquire`和`release`操作会同时执行到具体的每个锁

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)