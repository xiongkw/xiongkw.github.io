---
layout: post
title: Curator-recipes Cache入门
categories: [编程, java]
tags: [curator, zookeeper, cache]
---

> `Curator Cache`会把指定zk路径下的所有子节点及其数据缓存到本地，这样一来，应用程序只需要读取本地缓存，而不需要每次都实时从zk服务端读取

#### 1. PathChildrenCache
`PathChildrenCache`用于缓存和监听指定路径下所有子节点(只包括子辈，不包括孙辈)的创建、删除和更新(`add、del、set`)事件

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();

    PathChildrenCache pathChildrenCache = new PathChildrenCache(curator, "/samples/caches", true);
    pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {
        @Override
        public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
            System.out.println(event.getType());
        }
    });
    pathChildrenCache.start();

    BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
    in.readLine();
    
    //省略close操作
}
```

> 在`zookeeper api`中，`watcher`只会触发一次，所以如果需要持续监听一个节点，则需要重复注册`watcher`，而在`curator`中只需要注册一次

> 通过`cli`连接zk服务端，并在节点`/samples/caches`下添加删除节点，`PathChildrenCacheListener`会收到相应通知

#### 2. NodeCache
`NodeCache`用于缓存和监听指定路径的`data`更新事件

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();

    NodeCache nodeCache = new NodeCache(curator, "/samples/caches");
    nodeCache.getListenable().addListener(new NodeCacheListener() {
        @Override
        public void nodeChanged() throws Exception {
            System.out.println(new String(nodeCache.getCurrentData().getData()));
        }
    });
    nodeCache.start();

    BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
    in.readLine();
}
```

> 监听指定节点下的data，当data改变时触发事件

#### 3. TreeCache
`TreeCache`用于缓存和监听指定路径下所有子节点和孙子节点以及节点数据的事件，它是`PathChildrenCache`和`NodeCache`的结合体

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();

    TreeCache treeCache = new TreeCache(curator, "/samples/caches");
    treeCache.getListenable().addListener(new TreeCacheListener() {
        @Override
        public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
            System.out.println(event.getType());
        }
    });
    treeCache.start();

    BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
    in.readLine();
}
```

> 不仅能监听`/samples/caches`的直接子节点，还能监听其孙子节点(例如`/samples/caches/aa/bb/cc`)，及其`data`

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)