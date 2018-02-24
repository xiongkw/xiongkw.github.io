---
layout: post
title: 源码分析ReentrantLock
categories: [编程, java]
tags: [lock, reentrant, 多线程]
---

> `jdk5`中增加了`Lock`的概念，线程同步又多了一种新的写法

#### 1. 概念
* `CAS`: `Compare And Swap`,即比较交换,我们常用的`乐观锁`就是这个原理
* `UnSafe`: 提供直接操作堆内存的方法，是`CAS`算法的一种实现
* `AQS`: `AbstractQueuedSynchronizer`，是`java.util.concurrent`的核心，`CountDownLatch、FutureTask、Semaphore、ReentrantLock`等都有一个内部类是这个抽象类的子类，`AQS`便是基于`CAS`和`UnSafe`实现

#### 2. 一个例子
```java
public class LockTest {
    private static Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            int finalI = i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println("Thread: " + finalI + " Try get lock");
                    lock.lock();
                    System.out.println("Thread: " + finalI + " Got lock");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        lock.unlock();
                    }
                }
            }).start();
        }
    }
}
```

运行结果
```
Thread: 0 Try get lock
Thread: 1 Try get lock
Thread: 2 Try get lock
Thread: 4 Try get lock
Thread: 0 Got lock
Thread: 3 Try get lock
Thread: 7 Try get lock
Thread: 6 Try get lock
Thread: 5 Try get lock
Thread: 8 Try get lock
Thread: 9 Try get lock
Thread: 1 Got lock
Thread: 2 Got lock
Thread: 4 Got lock
Thread: 3 Got lock
Thread: 7 Got lock
Thread: 6 Got lock
Thread: 5 Got lock
Thread: 8 Got lock
Thread: 9 Got lock
```

#### 3. 源码分析

##### 3.1 ReentrantLock
`ReentrantLock`构造函数
```java
public ReentrantLock() {
    sync = new NonfairSync();
}
    
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

> `ReentrantLock`默认使用`非公平锁`

##### 3.2 加锁

`ReentrantLock.lock()`
```java
public void lock() {
    sync.lock();
}
```

`NonfairSync.lock()`
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

> `compareAndSetState`使用`UnSafe`实现，用于更改同步器的状态   
> `setExclusiveOwnerThread`，更改状态成功后把当前线程设置为同步器的占用线程

##### 3.3 等待锁

`NonfairSync.acquire()`
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
         selfInterrupt();
 }
```

先看`NonfairSync.tryAcquire()`
```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

> `state == 0 `说明没有被占用，这里尝试占用，如果成功，便返回，同`NonfairSync.lock()`   
> `current == getExclusiveOwnerThread()`，说明已经被当前线程占用了，这里会设置新的`state`，可看成是计数，这便是`Reentrant`的实现

再看`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

> `addWaiter`是把没有获得锁的线程放入等待队列


`NonfairSync.acquireQueued`
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

> 这里有一个尝试占用的动作`p == head && tryAcquire(arg)`，如果成功则返回   
> `shouldParkAfterFailedAcquire(p, node)`: 判断线程是否需要被阻塞

`NonfairSync.parkAndCheckInterrupt`
```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

`LockSupport.park`调用了`UNSAFE.park`，其解释:
> Disables the current thread for thread scheduling purposes unless the permit is available.   
> If the permit is available then it is consumed and the call returns immediately; otherwise the current thread becomes disabled for thread scheduling purposes and lies dormant until one of three things happens:   
> Some other thread invokes unpark with the current thread as the target; or   
> Some other thread interrupts the current thread; or   
> The call spuriously (that is, for no reason) returns.   
> This method does not report which of these caused the method to return. Callers should re-check the conditions which caused the thread to park in the first place. Callers may also determine, for example, the interrupt status of the thread upon return.   

##### 3.4 释放锁

`ReentrantLock.unlock`
```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

> `unparkSuccessor(h)`如果锁释放成功，则会唤醒`head`节点

`NonfairSync.tryRelease`
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

> `if (c == 0)`: `Reentrant`的释放

