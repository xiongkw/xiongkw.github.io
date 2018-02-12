---
layout: post
title: Curator-recipes Election源码分析
categories: [编程, java]
tags: [curator, zookeeper, election]
---

> 这里的选举指的是客户端的选举，而`zab`协议中的选举说是的`zk`服务端实例的选举

#### 1. 一个例子

```java
public static void main(String[] args) throws InterruptedException, IOException {
    LeaderLatch[] leaderLatches = new LeaderLatch[5];
    for (int i = 0; i < 5; i++) {
        int finalI = i;
        new Thread(() -> {
            CuratorFramework curator = CuratorFrameworkFactory.builder().retryPolicy(new RetryUntilElapsed(1000, 6000)).connectString("127.0.0.1:2181").build();
            curator.start();
            leaderLatches[finalI] = new LeaderLatch(curator, "/samples/leader", "node: "+finalI);
            try {
                leaderLatches[finalI].start();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }
    Thread.sleep(5000);

    for (LeaderLatch leaderLatch : leaderLatches) {
        if (leaderLatch.hasLeadership()) {
            System.out.println("Leader is: " + leaderLatch.getId());
            leaderLatch.close();
            break;
        }
    }

    Thread.sleep(5000);
    for (LeaderLatch leaderLatch : leaderLatches) {
        if (leaderLatch.hasLeadership()) {
            System.out.println("Leader is: " + leaderLatch.getId());
        }
    }
}
```

#### 2. 源码分析
`LeaderLatch#start`
```java
public void start() throws Exception {
    Preconditions.checkState(state.compareAndSet(State.LATENT, State.STARTED), "Cannot be started more than once");

    startTask.set(AfterConnectionEstablished.execute(client, new Runnable() {
        @Override
        public void run() {
            try {
                internalStart();
            } finally {
                startTask.set(null);
            }
        }
    }));
}

private synchronized void internalStart() {
    if ( state.get() == State.STARTED ) {
        client.getConnectionStateListenable().addListener(listener);
        try {
            reset();
        } catch ( Exception e ) {
            ThreadUtils.checkInterrupted(e);
            log.error("An error occurred checking resetting leadership.", e);
        }
    }
}

void reset() throws Exception {
    setLeadership(false);
    setNode(null);

    BackgroundCallback callback = new BackgroundCallback() {
        @Override
        public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {
            if ( debugResetWaitLatch != null ) {
                debugResetWaitLatch.await();
                debugResetWaitLatch = null;
            }

            if ( event.getResultCode() == KeeperException.Code.OK.intValue() ) {
                setNode(event.getName());
                if ( state.get() == State.CLOSED ) {
                    setNode(null);
                } else {
                    getChildren();
                }
            } else {
                log.error("getChildren() failed. rc = " + event.getResultCode());
            }
        }
    };
    client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).inBackground(callback).forPath(ZKPaths.makePath(latchPath, LOCK_NAME), LeaderSelector.getIdBytes(id));
}
```

> 选举开始时，每个客户端都会在zk指定路径创建一个有序临时节点

`getChildren`
```java
private void getChildren() throws Exception
{
    BackgroundCallback callback = new BackgroundCallback()
    {
        @Override
        public void processResult(CuratorFramework client, CuratorEvent event) throws Exception
        {
            if ( event.getResultCode() == KeeperException.Code.OK.intValue() )
            {
                checkLeadership(event.getChildren());
            }
        }
    };
    client.getChildren().inBackground(callback).forPath(ZKPaths.makePath(latchPath, null));
}
```

`checkLeadership`
```java
private void checkLeadership(List<String> children) throws Exception {
    final String localOurPath = ourPath.get();
    List<String> sortedChildren = LockInternals.getSortedChildren(LOCK_NAME, sorter, children);
    int ourIndex = (localOurPath != null) ? sortedChildren.indexOf(ZKPaths.getNodeFromPath(localOurPath)) : -1;
    if ( ourIndex < 0 ) {
        log.error("Can't find our node. Resetting. Index: " + ourIndex);
        reset();
    } else if ( ourIndex == 0 ) {
        setLeadership(true);
    } else {
        String watchPath = sortedChildren.get(ourIndex - 1);
        Watcher watcher = new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if ( (state.get() == State.STARTED) && (event.getType() == Event.EventType.NodeDeleted) && (localOurPath != null) ) {
                    try {
                        getChildren();
                    } catch ( Exception ex ) {
                        ThreadUtils.checkInterrupted(ex);
                        log.error("An error occurred checking the leadership.", ex);
                    }
                }
            }
        };

        BackgroundCallback callback = new BackgroundCallback() {
            @Override
            public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {
                if ( event.getResultCode() == KeeperException.Code.NONODE.intValue() ) {
                    // previous node is gone - reset
                    reset();
                }
            }
        };
        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
        client.getData().usingWatcher(watcher).inBackground(callback).forPath(ZKPaths.makePath(latchPath, watchPath));
    }
}
```

> 这里的选举逻辑和`InterProcessMutex`的获取逻辑类似，序号最小的节点成为`Leader`，其它节点则在其前一个序号的节点注册监听器，前一个序号节点删除后才会进行新一轮选举

#### 3. LeaderSelector

`LeaderSelector.doWork`
```java
void doWork() throws Exception {
    hasLeadership = false;
    try {
        mutex.acquire();

        hasLeadership = true;
        // ...
        listener.takeLeadership(client);
        
        // ...
    } finally {
        if ( hasLeadership ) {
            hasLeadership = false;
            // ...
            mutex.release();
            // ...
        }
    }
}
```
> `LeaderSelector`实际是使用`InterProcessMutex`锁来竞争，谁获得锁，谁就是`Leader`，与`LeaderLatch`不同的是`LeaderSelector`在`listener.takeLeadership(client)`中可以控制是否释放领导权，而`LeaderLatch`则是`Leader`挂了才会释放

#### 4. 参考文档

[ZooKeeper Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.8/recipes.html)

[Curator-recipes Barrier](http://curator.apache.org/curator-recipes/barrier.html)