---
layout: post
title: Curator-recipes Counter源码分析
categories: [编程, java]
tags: [curator, zookeeper, counter]
---

> 在同`jvm`进程内的多线程环境，我们通常使用`AtomicInteger`来实现计数器

#### 1. 一个例子

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();

    SharedCount sharedCount = new SharedCount(curator, "/samples/counters", 0);
    sharedCount.addListener(new SharedCountListener() {
        @Override
        public void countHasChanged(SharedCountReader sharedCount, int newCount) throws Exception {
            System.out.println(newCount);
        }

        @Override
        public void stateChanged(CuratorFramework client, ConnectionState newState) {

        }
    });
    sharedCount.start();

    sharedCount.setCount(10);

    sharedCount.trySetCount(sharedCount.getVersionedValue(), 10);
}
```
> `SharedCountListener`能够监听到`count`的变化   

#### 2. 源码分析
创建计数器
```java
public void start() throws Exception{
    Preconditions.checkState(state.compareAndSet(State.LATENT, State.STARTED), "Cannot be started more than once");

    client.getConnectionStateListenable().addListener(connectionStateListener);
    try {
        client.create().creatingParentContainersIfNeeded().forPath(path, seedValue);
    } catch ( KeeperException.NodeExistsException ignore ) {
        // ignore
    }

    readValue();
}
```

> 创建计数器实际是在zk指定路径创建一个节点，并写入初始计数值

`setCount`
```java
public void     setCount(int newCount) throws Exception{
    sharedValue.setValue(toBytes(newCount));
}
```

`SharedValue.setValue`
```java
public void setValue(byte[] newValue) throws Exception{
    Preconditions.checkState(state.get() == State.STARTED, "not started");

    Stat result = client.setData().forPath(path, newValue);
    updateValue(result.getVersion(), Arrays.copyOf(newValue, newValue.length));
}
```

> 直接写入新的数值到节点

`trySetCount`
```java
public boolean  trySetCount(VersionedValue<Integer> previous, int newCount) throws Exception{
    VersionedValue<byte[]> previousCopy = new VersionedValue<byte[]>(previous.getVersion(), toBytes(previous.getValue()));
    return sharedValue.trySetValue(previousCopy, toBytes(newCount));
}
```
`SharedValue.trySetValue`
```java
public boolean trySetValue(VersionedValue<byte[]> previous, byte[] newValue) throws Exception {
    Preconditions.checkState(state.get() == State.STARTED, "not started");

    VersionedValue<byte[]> current = currentValue.get();
    if ( previous.getVersion() != current.getVersion() || !Arrays.equals(previous.getValue(), current.getValue()) ) {
        return false;
    }

    try {
        Stat result = client.setData().withVersion(previous.getVersion()).forPath(path, newValue);
        updateValue(result.getVersion(), Arrays.copyOf(newValue, newValue.length));
        return true;
    } catch ( KeeperException.BadVersionException ignore ) {
        // ignore
    }

    readValue();
    return false;
}
```

> `trySetCount`与`setCount`的不同之处在于前者使用了节点数据的版本号作为乐观锁

#### 3. 其它计数器

* `DistributedAtomicLong` `Long`类型的计数器，使用了分布式锁`InterProcessMutex`来保证操作的原子性

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)