---
layout: post
title: Java中Object.wait和Thread.sleep
categories: [编程, java]
tags: [wait, notify, 多线程, sleep]
---

> 多线程编码中，我们常常使用`wait`或`sleep`来让线程`暂停`一段时间，那么这两种到底有什么区别呢？

#### 1. 一个wait的例子

代码：
```java
public static void main(String[] args) throws InterruptedException {
    Object lock = new Object();
    new Thread(() -> {
        try {
            Thread.sleep(1000);

            synchronized (lock) {
                System.out.println("Thread got lock");
                lock.notifyAll();
                Thread.sleep(2000);
                System.out.println("Thread end");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();

    synchronized (lock) {
        System.out.println("Main got lock");
        lock.wait();
        System.out.println("Main got lock");
        System.out.println("Main end");
    }

    while (Thread.activeCount() > 2) {
        Thread.yield();
    }
    System.out.println("End");
}
```

运行结果：
```
Main got lock
Thread got lock
Thread end
Main got lock
Main end
End
```

> 可以看到`Main`在`wait`的时候释放了锁，而当`Thread`结束之后，`Main`又重新获得了锁

#### 2. 一个sleep的例子

代码：
```java
public static void main(String[] args) throws InterruptedException {
    Object lock = new Object();
    new Thread(() -> {
        try {
            Thread.sleep(1000);

            synchronized (lock) {
                System.out.println("Thread got lock");
                Thread.sleep(2000);
                System.out.println("Thread end");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();

    synchronized (lock) {
        System.out.println("Main got lock");
        Thread.sleep(2000);
        System.out.println("Main end");
    }

    while (Thread.activeCount() > 2) {
        Thread.yield();
    }
    System.out.println("End");
}
```

运行结果：
```
Main got lock
Main end
Thread got lock
Thread end
End
```

> 可以看到`Main`在`sleep`的过程中并没有释放锁，而直到`Main`结束之后，`Thread`才能获取到锁

#### 3. 总结

* `wait`是`Object`对象的方法，其调用一定要发生在获取该对象锁(`synchronized(lock)`)之后
* `sleep`是`Thread`类的方法，其调用通常与锁无关
* `wait`方法会释放锁，而`sleep`不会