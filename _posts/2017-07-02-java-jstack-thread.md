---
layout: post
title: 使用jstack分析java线程状态
categories: [编程, java]
tags: [thread, jstack]
---

#### 1. 一个例子
```java
public class JstackDemo {
    public static void main(String[] args) {
        Object l1 = new Object();

        Thread t1 = new Thread(() -> {
            synchronized (l1) {
                try {
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        t1.setName("Thread 1");
        t1.start();

        Thread t2 = new Thread(() -> {
            synchronized (l1) {
            }
        });
        t2.setName("Thread 2");
        t2.start();
    }
}
```

> 上述代码使用两个线程同时竞争一个锁演示线程阻塞情况

#### 2. 获取线程dump

1. 使用`jps`命令查找进程`id`

```
jps

39360 JstackDemo
```

2. 使用`jstack`命令获取线程`dump`

```
jstack 39260

"Thread 2" #12 prio=5 os_prio=0 tid=0x14e4d800 nid=0xb378 waiting for monitor entry [0x15c0f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.mywork.lock.JstackDemo.lambda$main$1(JstackDemo.java:24)
        - waiting to lock <0x0473fd88> (a java.lang.Object)
        at com.mywork.lock.JstackDemo$$Lambda$2/24759020.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

"Thread 1" #11 prio=5 os_prio=0 tid=0x14e4cc00 nid=0xb3cc waiting on condition [0x15a9f000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at com.mywork.lock.JstackDemo.lambda$main$0(JstackDemo.java:13)
        - locked <0x0473fd88> (a java.lang.Object)
        at com.mywork.lock.JstackDemo$$Lambda$1/31498271.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
```

* `Thread 1`调用了`Thread.sleep(100000);`，所以其状态为`TIMED_WAITING`
* `Thread 2`在等待锁，所以其状态为`BLOCKED`

#### 3. Thread Dump说明

```
"Thread 2" #12 prio=5 os_prio=0 tid=0x14e4d800 nid=0xb378 waiting for monitor entry [0x15c0f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.mywork.lock.JstackDemo.lambda$main$1(JstackDemo.java:24)
        - waiting to lock <0x0473fd88> (a java.lang.Object)
        at com.mywork.lock.JstackDemo$$Lambda$2/24759020.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
```

* `Thread 2`: 线程名称
* `prio=5`: 优先级
* `tid=0x14e4d800`: 线程`id(jvm)`
* `nid=0xb378`: 本地线程`id`
* `java.lang.Thread.State`: 线程状态

线程状态：

* `TIMED_WAITING`:线程调用了`wait(long)`或者`join(long)`或`sleep(long)`的情况下
* `WAITING`:等待中，线程获得一个对象锁后，在该对象上等待其他线程来`notify()`
* `BLOCKED`:被阻塞，线程在等待临界资源被释放，比如等待另外一个线程释放临界资源
* `RUNNABLE`:可运行状态，当获得`CPU`的使用权时就可运行的线程

#### 4. 死锁的例子

##### 4.1 代码

```java
public class DeadLock {
    public static void main(String[] args) {
        Object l1 = new Object();
        Object l2 = new Object();

        Thread t1 = new Thread(() -> {
            synchronized (l1) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (l2) {
                    System.out.println("haha");
                }
            }
        });
        t1.setName("Thread 1");
        t1.start();

        Thread t2 = new Thread(() -> {
            synchronized (l2) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (l1) {
                    System.out.println("haha");
                }
            }
        });
        t2.setName("Thread 2");
        t2.start();
    }
}

```

##### 4.2 Thread Dump

```
Full thread dump Java HotSpot(TM) Client VM (25.51-b03 mixed mode):

"DestroyJavaVM" #13 prio=5 os_prio=0 tid=0x0032e800 nid=0xaa50 waiting on condition [0x00000000]
   java.lang.Thread.State: RUNNABLE

"Thread 2" #12 prio=5 os_prio=0 tid=0x14fbf000 nid=0x9a94 waiting for monitor entry [0x0271f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.mywork.lock.DeadLock.lambda$main$1(DeadLock.java:34)
        - waiting to lock <0x049400b0> (a java.lang.Object)
        - locked <0x049400b8> (a java.lang.Object)
        at com.mywork.lock.DeadLock$$Lambda$2/24759020.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

"Thread 1" #11 prio=5 os_prio=0 tid=0x14fbe400 nid=0xb3c4 waiting for monitor entry [0x1588f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.mywork.lock.DeadLock.lambda$main$0(DeadLock.java:19)
        - waiting to lock <0x049400b8> (a java.lang.Object)
        - locked <0x049400b0> (a java.lang.Object)
        at com.mywork.lock.DeadLock$$Lambda$1/31498271.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

...

Found one Java-level deadlock:
=============================
"Thread 2":
  waiting to lock monitor 0x0264db6c (object 0x049400b0, a java.lang.Object),
  which is held by "Thread 1"
"Thread 1":
  waiting to lock monitor 0x0264f6fc (object 0x049400b8, a java.lang.Object),
  which is held by "Thread 2"

Java stack information for the threads listed above:
===================================================
"Thread 2":
        at com.mywork.lock.DeadLock.lambda$main$1(DeadLock.java:34)
        - waiting to lock <0x049400b0> (a java.lang.Object)
        - locked <0x049400b8> (a java.lang.Object)
        at com.mywork.lock.DeadLock$$Lambda$2/24759020.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
"Thread 1":
        at com.mywork.lock.DeadLock.lambda$main$0(DeadLock.java:19)
        - waiting to lock <0x049400b8> (a java.lang.Object)
        - locked <0x049400b0> (a java.lang.Object)
        at com.mywork.lock.DeadLock$$Lambda$1/31498271.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.
```

> 可以看到`Thread 1`和`Thread 2`在竞争锁的过程中发生了死锁，其状态都是`BLOCKED`