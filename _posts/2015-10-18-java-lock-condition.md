---
layout: post
title: Java Lock之Condition
categories: [编程, java]
tags: [lock, condition, 多线程]
---

> 在多线程编程中我们常常使用`Object`的`wait`和`notify`配合`synchronized`来实现有条件的锁，本文演示如何使用`Condition`实现同样的功能

代码：

```java
public class ConditionTest extends Thread {
    private static Lock lock = new ReentrantLock();
    private static Condition c = lock.newCondition();
    private static AtomicInteger ai = new AtomicInteger();

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    lock.lock();
                    try {
                        Thread.sleep(100);
                        int i = LockTest.ai.incrementAndGet();
                        c.signalAll();
                        if (i < 5) {
                            System.out.println("Counter: " + i);
                            c.await();
                        }
                        System.out.println("More than 5 threads, continue");
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

> 这里使用`Condition`实现当启动线程总数达到`5`时才继续运行，否则每个线程都等待

运行结果
```
Counter: 1
Counter: 2
Counter: 3
Counter: 4
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
More than 5 threads, continue
```