---
layout: post
title: Curator-recipes Nodes源码分析
categories: [编程, java]
tags: [curator, zookeeper, node, group]
---

> 在分布式系统中，我们通常使用临时节点来代表某个客户端的在线离线状态，使用zk原生api则要编码实现断线后重新注册，而`Curator`已经实现了类似的功能`PersistentNode`

#### 1. 一个例子

```java
public static void main(String[] args) throws Exception {
    CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
    curator.start();

    PersistentNode node = new PersistentNode(curator, CreateMode.EPHEMERAL_SEQUENTIAL, true, "/samples/nodes/node", "hello".getBytes());
    node.getListenable().addListener(new PersistentNodeListener() {
        @Override
        public void nodeCreated(String path) throws Exception {
            String actualPath = node.getActualPath();
            Stat stat = curator.checkExists().forPath(actualPath);
            System.out.println(stat);

            Thread.sleep(4000);
            System.out.println(curator.checkExists().forPath(actualPath));
        }
    });
    node.start();

    BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
    in.readLine();
}
```

#### 2. 源码分析
构造函数
```java
public PersistentNode(CuratorFramework givenClient, final CreateMode mode, boolean useProtection, final String basePath, byte[] initData, long ttl) {
    this.useProtection = useProtection;
    this.client = Preconditions.checkNotNull(givenClient, "client cannot be null").newWatcherRemoveCuratorFramework();
    this.basePath = PathUtils.validatePath(basePath);
    this.mode = Preconditions.checkNotNull(mode, "mode cannot be null");
    this.ttl = ttl;
    final byte[] data = Preconditions.checkNotNull(initData, "data cannot be null");

    backgroundCallback = new BackgroundCallback() {
        @Override
        public void processResult(CuratorFramework dummy, CuratorEvent event) throws Exception {
            if ( isActive() ) {
                processBackgroundCallback(event);
            } else {
                processBackgroundCallbackClosedState(event);
            }
        }
    };

    this.data.set(Arrays.copyOf(data, data.length));
}
```

> `useProtection`说的是在某些极端的情况下，例如创建一个有序节点时，因为节点名称在zk服务端生成，所以客户端需要收到zk服务端的响应才知道节点的名称，而极端原因导致客户端没有收到，这样客户端便无法知道自己创建节点的名称。

> It turns out there is an edge case that exists when creating sequential-ephemeral nodes. The creation can succeed on the server, but the server can crash before the created node name is returned to the client. However, the ZK session is still valid so the ephemeral node is not deleted. Thus, there is no way for the client to determine what node was created for them.   
> Even without sequential-ephemeral, however, the create can succeed on the sever but the client (for various reasons) will not know it.   
> Putting the create builder into protection mode works around this. The name of the node that is created is prefixed with a GUID. If node creation fails the normal retry mechanism will occur. On the retry, the parent path is first searched for a node that has the GUID in it. If that node is found, it is assumed to be the lost node that was successfully created on the first try and is returned to the caller.

PersistentNode.start
```java
public void start() {
    Preconditions.checkState(state.compareAndSet(State.LATENT, State.STARTED), "Already started");

    client.getConnectionStateListenable().addListener(connectionStateListener);
    createNode();
}

private final ConnectionStateListener connectionStateListener = new ConnectionStateListener() {
    @Override
    public void stateChanged(CuratorFramework dummy, ConnectionState newState) {
        if ( (newState == ConnectionState.RECONNECTED) && isActive() ) {
            createNode();
        }
    }
};
```

> 使用`ConnectionStateListener`监听重连事件，并重新创建节点

PersistentNode.createNode
```java
private void createNode() {
    if ( !isActive() ) {
        return;
    }

    if ( debugCreateNodeLatch != null ) {
        try {
            debugCreateNodeLatch.await();
        } catch ( InterruptedException e ) {
            Thread.currentThread().interrupt();
            return;
        }
    }

    try {
        String existingPath = nodePath.get();
        String createPath = (existingPath != null && !useProtection) ? existingPath : basePath;

        CreateModable<ACLBackgroundPathAndBytesable<String>> localCreateMethod = createMethod.get();
        if ( localCreateMethod == null ) {
            CreateBuilderMain createBuilder = SafeIsTtlMode.isTtl(mode) ? client.create().withTtl(ttl) : client.create();
            CreateModable<ACLBackgroundPathAndBytesable<String>> tempCreateMethod = useProtection ? createBuilder.creatingParentContainersIfNeeded().withProtection() : createBuilder.creatingParentContainersIfNeeded();
            createMethod.compareAndSet(null, tempCreateMethod);
            localCreateMethod = createMethod.get();
        }
        localCreateMethod.withMode(getCreateMode(existingPath != null)).inBackground(backgroundCallback).forPath(createPath, data.get());
    } catch ( Exception e ) {
        ThreadUtils.checkInterrupted(e);
        throw new RuntimeException("Creating node. BasePath: " + basePath, e);  // should never happen unless there's a programming error - so throw RuntimeException
    }
}

```

PersistentNode.getCreateMode
```java
private CreateMode getCreateMode(boolean pathIsSet) {
    if ( pathIsSet ) {
        switch ( mode ) {
            default:
            {
                break;
            }

            case EPHEMERAL_SEQUENTIAL:
            {
                return CreateMode.EPHEMERAL;    // protection case - node already set
            }

            case PERSISTENT_SEQUENTIAL:
            {
                return CreateMode.PERSISTENT;    // protection case - node already set
            }

            case PERSISTENT_SEQUENTIAL_WITH_TTL:
            {
                return CreateMode.PERSISTENT_WITH_TTL;    // protection case - node already set
            }
        }
    }
    return mode;
}
```
> 如果节点曾经创建成功过，则这里会把相应`SEQUENTIAL`去掉，保证重新创建的节点名不会变化，这里是针对有序节点，因为无序节点的名称是固定的

PsersistentNode.watchNode
```java
private void watchNode() throws Exception {
    if ( !isActive() ) {
        return;
    }

    String localNodePath = nodePath.get();
    if ( localNodePath != null ) {
        client.checkExists().usingWatcher(watcher).inBackground(checkExistsCallback).forPath(localNodePath);
    }
}

private final BackgroundCallback checkExistsCallback = new BackgroundCallback() {
    @Override
    public void processResult(CuratorFramework dummy, CuratorEvent event) throws Exception {
        if ( isActive() ) {
            if ( event.getResultCode() == KeeperException.Code.NONODE.intValue() ) {
                createNode();
            } else {
                boolean isEphemeral = event.getStat().getEphemeralOwner() != 0;
                if ( isEphemeral != mode.isEphemeral() ) {
                    log.warn("Existing node ephemeral state doesn't match requested state. Maybe the node was created outside of PersistentNode? " + basePath);
                }
            }
        } else {
            client.removeWatchers();
        }
    }
};

```

> 监听节点状态，如果节点被删除了，这里会重新创建

#### 3. PersistentTtlNode

`PersistentTtlNode`提供生命周期的功能，类似缓存，和`PersistentNode`不同的是`PersistentTtlNode`只支持持久节点

#### 4. GroupMember

`GroupMember` 用来创建一个分组，其构造构造函数为
```java
public GroupMember(CuratorFramework client, String membershipPath, String thisId, byte[] payload) {
    this.thisId = Preconditions.checkNotNull(thisId, "thisId cannot be null");

    cache = newPathChildrenCache(client, membershipPath);
    pen = newPersistentEphemeralNode(client, membershipPath, thisId, payload);
}
```
> 通过`membershipPath`指定组名(或者说标识), `thisId`为当前客户端的ID

```java
protected PersistentEphemeralNode newPersistentEphemeralNode(CuratorFramework client, String membershipPath, String thisId, byte[] payload) {
    return new PersistentEphemeralNode(client, PersistentEphemeralNode.Mode.EPHEMERAL, ZKPaths.makePath(membershipPath, thisId), payload);
}
```

> 这里分创建一个路径为`membershipPath/thisId`的临时节点


使用`getCurrentMembers`方法获取当前组员
```java
public Map<String, byte[]> getCurrentMembers() {
    ImmutableMap.Builder<String, byte[]> builder = ImmutableMap.builder();
    boolean thisIdAdded = false;
    for ( ChildData data : cache.getCurrentData() ) {
        String id = idFromPath(data.getPath());
        thisIdAdded = thisIdAdded || id.equals(thisId);
        builder.put(id, data.getData());
    } if ( !thisIdAdded ) {
        builder.put(thisId, pen.getData());   // this instance is always a member
    }
    return builder.build();
}
```
> 返回的是组员ID->data的Map，组员总是包括自己

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)