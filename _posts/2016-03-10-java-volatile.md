---
layout: post
title: java中的volatile
categories: [编程, java]
tags: [volatile, 多线程]
---

> java中使用volatile关键字可以解决一定程度的多线程并发问题 

#### 1. java内存模型
> java栈用来保存线程运行时的状态，包括变量值(基本类型的值，对象的引用地址)，这些值或引用地址是保存在线程的私有栈中.   
> 也就是说每个线程都会保存一个副本，而其原值则保存在堆中(共享内存区)

#### 1. 例子

这里使用一个读线程循环读取`t.b`的值，主线程则改写其值
```java
public class VolatileTest {
    private  boolean b = false;

    public static void main(String[] args) throws InterruptedException {
    	final VolatileTest t = new VolatileTest();
    	new Thread(new Runnable() {
            public void run() {
                while (!t.b);
            }
        }).start();
    	Thread.sleep(100);
        t.b = true;
	}
}
```

> 这段代码在jdk7中运行时不会退出，但在jdk8中却会正常退出(猜想可能是jdk8做了一些优化)

解释：读操作时jvm从堆中读取`t.b`值并写入到方法栈中，而写操作则是把方法栈中的值写入到堆中`t.b`。因为把方法栈中的值写入堆中的时间是不确定的，所以读线程无法读取到最新的值。

#### 2. 使用volatile

改进的代码
```java
public class VolatileTest {
    // 这里使用`volatile`关键字后，程序能正常退出了
    private volatile boolean b = false;

    // ...        
}
```

> `volatile`的作用是告诉jvm对`t.b`的读操作每次从堆内存读取最新的，而写操作则是每次都及时写入堆内存。

#### 3. 线程安全

```java
public class VolatileTest2 {
    private volatile int i = 0;
    private AtomicInteger ai = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        VolatileTest2 t = new VolatileTest2();
        for (int i = 0; i < 10; i++)
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    t.i++;
                    t.ai.incrementAndGet();
                }
            }).start();

        Thread.sleep(3000);
        System.out.println(t.i);
        System.out.println(t.ai.get());
    }

}
```

运行结果
```
9600
10000
```

> 为什么i的值不是10000?这是因为java中的`i++`不是线程安全的，从内存模型上看`t.i++`分为三个步骤：   
> 1. 从堆内存读取t.i值到栈中   
> 2. 对栈中的i值副本+1   
> 3. 把i值副本写回堆内存`t.i`

刚才发生了什么？ 通过`volatile`关键字，线程虽然能够读取最新的`t.i`的值，但是是写的过程中冲突了。而`AtomicInteger`保证其自增操作是一个原子操作，所以是线程安全的。

#### 4. 关于atomic

参考[java中的atomic]({{ site.url}}/2016/03/12/java-atomic/)