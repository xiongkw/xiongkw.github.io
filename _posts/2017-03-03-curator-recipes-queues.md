---
layout: post
title: Curator-recipes Queue源码分析
categories: [编程, java]
tags: [curator, zookeeper, queue]
---

> `Queue`是一种数据结构,它有两个基本操作：在队列尾部加入一个元素，和从队列头部移除一个元素，队列以一种先进先出的方式管理数据

#### 1. 一个例子

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();
    QueueBuilder<String> builder = QueueBuilder.builder(curator, new QueueConsumer<String>() {
        @Override
        public void consumeMessage(String message) throws Exception {
            System.out.println(message);
        }

        @Override
        public void stateChanged(CuratorFramework client, ConnectionState newState) {

        }
    }, new QueueSerializer<String>() {
        @Override
        public byte[] serialize(String item) {
            return item.getBytes();
        }

        @Override
        public String deserialize(byte[] bytes) {
            return new String(bytes);
        }
    }, "/samples/queues");
    DistributedQueue<String> queue = builder.buildQueue();
    queue.start();

    for (int i = 0; i < 10; i++) {
        queue.put("msg:" + i);
    }

    Thread.sleep(3000);
    
    //省略close操作
}
```

运行结果
```
msg:0
msg:1
msg:2
msg:3
msg:4
msg:5
msg:6
msg:7
msg:8
msg:9
```

#### 2. 源码分析

队列的创建
```java
public void     start() throws Exception {
    if ( !state.compareAndSet(State.LATENT, State.STARTED) ) {
        throw new IllegalStateException();
    }

    try {
        client.create().creatingParentContainersIfNeeded().forPath(queuePath);
    }
    
    // ...
 }
```

> 队列的创建实际是创建了一个zk节点

写入队列
```java
public boolean     put(T item, int maxWait, TimeUnit unit) throws Exception {
    checkState();

    String      path = makeItemPath();
    return internalPut(item, null, path, maxWait, unit);
}
```

`internalCreateNode()`方法
```java
void internalCreateNode(String path, byte[] bytes, BackgroundCallback callback) throws Exception
{
    client.create().withMode(CreateMode.PERSISTENT_SEQUENTIAL).inBackground(callback).forPath(path, bytes);
}
```

> 写入队列实际是在zk创建了一个有序持久节点，其data为序列化后的字节

队列的消费
```java
public void     start() throws Exception {
    // ...
    service.submit(new Callable<Object>(){
            @Override
            public Object call(){
                runLoop();
                return null;
            }
        }
    );
 }
```

`runLoop()`
```java
while ( state.get() == State.STARTED  ){
    //...
    processChildren(children, currentVersion);
    //...
}
```

`processNormally()`
```java
private boolean processNormally(String itemNode, ProcessType type) throws Exception {
    try {
        String  itemPath = ZKPaths.makePath(queuePath, itemNode);
        Stat    stat = new Stat();

        byte[]  bytes = null;
        if ( type == ProcessType.NORMAL ) {
            bytes = client.getData().storingStatIn(stat).forPath(itemPath);
        }
        if ( client.getState() == CuratorFrameworkState.STARTED ) {
            client.delete().withVersion(stat.getVersion()).forPath(itemPath);
        }

        if ( type == ProcessType.NORMAL ) {
            processMessageBytes(itemNode, bytes);
        }

        return true;
    }
}
```

> 消费一个`item`时会删除对应节点

`processMessageBytes`
```java
private ProcessMessageBytesCode processMessageBytes(String itemNode, byte[] bytes) throws Exception {
    ProcessMessageBytesCode     resultCode = ProcessMessageBytesCode.NORMAL;
    MultiItem<T>                items;
    try {
        items = ItemSerializer.deserialize(bytes, serializer);
    }
    catch ( Throwable e ) {
        ThreadUtils.checkInterrupted(e);
        log.error("Corrupted queue item: " + itemNode, e);
        return resultCode;
    }

    for(;;) {
        T       item = items.nextItem();
        if ( item == null ) {
            break;
        }

        try {
            consumer.consumeMessage(item);
        }
        catch ( Throwable e ) {
            ThreadUtils.checkInterrupted(e);
            log.error("Exception processing queue item: " + itemNode, e);
            if ( errorMode.get() == ErrorMode.REQUEUE ) {
                resultCode = ProcessMessageBytesCode.REQUEUE;
                break;
            }
        }
    }
    return resultCode;
}
```

> 消费item

#### 3. 其它队列

* `DistributedIdQueue`，带有id的队列(类似`map`)，其`put`方法`put(T item, String itemId)`，同时也提供了`remove`方法`remove(String id)`，即可在消费之前从队列中删除(后悔药)
* `DistributedPriorityQueue`，队列中的元素按照优先级进行排序, `Priority`小的会优先被消费
* `DistributedDelayQueue`，在指定`delay`时间过后，才可能被消费

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)