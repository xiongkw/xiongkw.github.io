---
layout: post
title: java中的atomic
categories: [编程, java]
tags: [atomic, 多线程]
---


> 我们常常使用`AtomicInteger`实现计数器
  
#### 1. CAS
`CAS`: `Compare And Swap`,即比较交换,我们常用的乐观锁就是这个原理

```
V表示要更新的变量
E表示预期值
CAS(V,E,N)
N表示新值
```

> `CAS`操作需要提供一个期望值，当期望值与当前线程的变量值相同时，说明还没线程修改该值，当前线程可以进行修改，但如果期望值与当前线程不符，则说明该值已被其他线程修改，此时不执行更新操作，也可以选择重新读取该变量再尝试再次修改该变量

#### 2. Unsafe

`Unsafe`: 存在于`sun.misc`包中，其内部方法操作可以像`C`的指针一样直接操作内存，不建议直接使用

#### 3. 一个例子
```java
public class AtomicIntegerTest {
    private volatile int i = 0;
    private AtomicInteger ai = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        AtomicIntegerTest t = new AtomicIntegerTest();
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

`AtomicInteger`的`incrementAndGet`方法是线程安全的

> 关于`volatile`，参考参考[java中的volatile]({{ site.url}}/2016/03/10/java-volatile/)

#### 4. 源码分析

`AtomicInteger.getAndIncrement`
```java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

`UnSafe.getAndAddInt`
```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

`Unsafe.compareAndSwapInt`
```java
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

> `Unsafe.compareAndSwapInt`是一个`native`方法，其实现便是`CAS`算法

#### 5. 总结

`AtomicXX`使用`Unsafe`类直接操作堆内存，通过轻量级比较交换算法(`CAS`)实现了无锁的线程安全，其并发性能优于同步锁