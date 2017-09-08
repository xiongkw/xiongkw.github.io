---
layout: post
title: Java并发中的CountDownLatch
categories: [编程, java]
tags: [并发, 多线程]
---


> 并发编程中需要等待多个线程完成以后才继续下一步，这时候就需要用到闭锁(CountDownLatch)

```java
        // 定义一个latch，锁的数量为2
        final CountDownLatch latch = new CountDownLatch(2);
        new Thread(){
            @Override
            public void run() {
                try {
                    //
                }  finally {
                    //释放锁，放在finally块中，以免线程抛出异常后，latch无限等待
                    latch.countDown();      
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                try {
                    //
                }  finally {
                    //释放锁，放在finally块中，以免线程抛出异常后，latch无限等待
                    latch.countDown();
                }
            }
        }.start();
        //等待所有锁释放完毕
        latch.await();
        System.out.println("All threads ended ");
```

> 注意latch.countDown()要放在finally块中，以免线程抛出异常后，latch无限等待