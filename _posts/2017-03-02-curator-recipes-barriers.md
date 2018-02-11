---
layout: post
title: Curator-recipes Barrier源码分析
categories: [编程, java]
tags: [curator, zookeeper, barrier]
---

> `Barrier` 即栅栏，在分布式系统中设置一个栅栏会阻塞所有节点上的进程，直到条件满足后再移除栅栏，然后所有的节点继续运行。比如赛马比赛中，等赛马陆续来到起跑线前，一声令下，所有的赛马都飞奔而出。

#### 1. 一个例子

以下代码使用多线程模拟分布式运行

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();
    DistributedBarrier barrier = new DistributedBarrier(curator, "/samples/barriers");
    // 设置一个栅栏，其节点路径为/samples/barriers
    barrier.setBarrier();
    System.out.println("Thread starting");
    for (int i = 0; i < 3; i++) {
        int finalI = i;
        new Thread(() -> {
            try {
                System.out.println("Thread " + finalI+" waiting");
                // 在栅栏前等待，这里线程被阻塞了
                barrier.waitOnBarrier();
                // 栅栏移除后，所有线程继续运行
                System.out.println("Thread " + finalI+" started");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }
    boolean someCondition = false;
    while(!someCondition) {
        Thread.sleep(2000);
        someCondition = true;
    }
    // 等待某条件满足后移除栅栏
    if(someCondition){
        barrier.removeBarrier();
    }
    System.out.println("All Thread starting");
    
    //省略close操作
}
```

运行结果
```
Thread starting
Thread 0 waiting
Thread 1 waiting
Thread 2 waiting

All Thread starting
Thread 1 started
Thread 2 started
Thread 0 started

```

#### 2. 源码分析
构造函数
```java
public DistributedBarrier(CuratorFramework client, String barrierPath) {
    this.client = client;
    this.barrierPath = PathUtils.validatePath(barrierPath);
}
```

> 构造函数传入`client`和`path`

设置栅栏
```java
public synchronized void setBarrier() throws Exception {
    try{
        client.create().creatingParentContainersIfNeeded().forPath(barrierPath);
    } catch ( KeeperException.NodeExistsException ignore ) {
        // ignore
    }
}
```

> 设置栅栏实际是在zk中创建了指定路径的节点

等待
```java
public synchronized boolean      waitOnBarrier(long maxWait, TimeUnit unit) throws Exception {
    long            startMs = System.currentTimeMillis();
    boolean         hasMaxWait = (unit != null);
    long            maxWaitMs = hasMaxWait ? TimeUnit.MILLISECONDS.convert(maxWait, unit) : Long.MAX_VALUE;

    boolean         result;
    for(;;) {
        result = (client.checkExists().usingWatcher(watcher).forPath(barrierPath) == null);
        if ( result ) {
            break;
        }

        if ( hasMaxWait ) {
            long        elapsed = System.currentTimeMillis() - startMs;
            long        thisWaitMs = maxWaitMs - elapsed;
            if ( thisWaitMs <= 0 ) {
                break;
            }
            wait(thisWaitMs);
        } else {
            wait();
        }
    }
    return result;
}
```

> 等待栅栏创建的节点被移除

移除栅栏
```java
public synchronized void removeBarrier() throws Exception {
    try {
        client.delete().forPath(barrierPath);
    } catch ( KeeperException.NoNodeException ignore )     {
        // ignore
    }
}
```

> 移除栅栏实际是删除zk节点

#### 3. 双栅栏
* `DistributedDoubleBarrier` 双栅栏不仅提供了同步启动，也提供了同步结束屏障的功能。

> 在java多线程中我们常常使用`CountDownLatch`来等待多个线程执行完成

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)