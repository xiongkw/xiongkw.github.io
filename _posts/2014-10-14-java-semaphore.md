---
layout: post
title: Java中的Semaphore
categories: [编程, java]
tags: [java, semaphore, 多线程]
---

> Semaphore: 一个计数信号量。从概念上讲，信号量维护了一个许可集。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个 release() 添加一个许可，从而可能释放一个正在阻塞的获取者。但是，不使用实际的许可对象，Semaphore 只对可用许可的号码进行计数，并采取相应的行动。拿到信号量的线程可以进入代码，否则就等待。通过acquire()和release()获取和释放访问许可。   \

下面代码用semaphore实现多线程的同步

```java
public class SemaphoreTest {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(1);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < 10; i++) {
            executorService.execute(() -> {
                String name = Thread.currentThread().getName();
                try {
                    semaphore.acquire();
                    System.out.println("Thread: " + name + " got the lock.");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    System.out.println("Thread: " + name + " released the lock");
                    semaphore.release();
                }
            });
        }
    }
}
```