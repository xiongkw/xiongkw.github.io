---
layout: post
title: java中的可见性、有序性和原子性
categories: [编程, java]
tags: [可见性, 有序性, 原子性, volatile, synchronized, 内存屏障, 多线程, cas]
---

#### 1. 简介
`java`内存模型：

![]({{ site.url}}/public/images/2016-06-28-java-visibility-ordering-atomic.png)

> 这里的线程工作内存也称为`stack`，即`栈`，主内存也称为`heap`，即`堆`

#### 2. 原子性

`Atomicity`, 原子性是指一个或者多个操作要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。常见的数据库`transaction`便是原子操作

java内存模型中的原子操作
* `lock (锁定)`：作用于主内存中的变量，它把一个变量标识为一个线程独占的状态
* `unlock (解锁)`：作用于主内存中的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
* `read (读取)`：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便后面的`load`动作使用
* `load (载入)`：作用于工作内存中的变量，它把`read`操作从主内存中得到的变量值放入工作内存中的变量副本
* `use (使用)`：作用于工作内存中的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作
* `assign (赋值)`：作用于工作内存中的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作
* `store (存储)`：作用于工作内存的变量，它把工作内存中一个变量的值传送给主内存中以便随后的`write`操作使用
* `write (操作)`：作用于主内存的变量，它把`store`操作从工作内存中得到的变量的值放入主内存的变量中

`java`中基本类型(`long/double`除外)的读写具有原子性，所以我们常常见到`volatile long`和`volatile double`的写法

> 关于`volatile long/double`，参考[java中long/double的非原子性和volatile]({{ site.url}}/2016/04/11/java-volatile-long-double/)

关于 `i++`，常见的`i++`是非原子操作，可分解为以下三个原子操作：
```
load(i);
assign(i+1);
store(i);
```

#### 3. 可见性

`Visibility`, 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。`java`中可以通过`synchronized(lock)`或`volatile`来实现可见性，其原理是在`临界区`前后插入`内存屏障`。

`CAS`, `Compare And Swap`，无锁交换算法的基础就是可见性

> 关于内存屏障，参考[java多线程中的内存屏障]({{ site.url}}/2016/04/19java-memory-barrier/)

> 类似的，数据库的隔离级别便是控制事务对共享数据的可见性

#### 4. 有序性

`Ordering`, 指程序执行的顺序按照代码的先后顺序执行。`java`中允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

例如以下代码：
```java
int i = 0;
int j = 0;
```

其真正执行顺序可能为：
```java
j=0;
i=0;
```

> 可通过`synchronized(lock)`或`volatile`保证有序性，参考[java多线程中的内存屏障]({{ site.url}}/2016/04/19java-memory-barrier/)
