---
layout: post
title: java多线程中的内存屏障
categories: [编程, java]
tags: [synchronized, volatile, memory barrier]
---

#### 1. 概念

`Memory Barrier`: 内存屏障由底层`cpu`指令实现，其作用是把一段指令序列隔离起来，使其之外(之前或之后)的指令无法穿越到其中

#### 内存屏障与重排序

> 关于`重排序`：任何一段高级程序代码最终都会以`指令序列`的形式被`cpu`执行，而`cpu`则会对指令序列重新排序，以获得最高的执行效率

通过成对的内存屏障能够隔离出一段指令序列，保证屏障内部的指令不会被重排到屏障外部，同时也保障屏障外部的指令不会进入屏障内部

但是，屏障内部的指令序列和其之前之后的指令序列仍然可能被重排序

#### 2. 内存屏障与可见性

既然内存屏障可以隔离一段指令序列，那其必然有两个严格的界限，内存屏障通过`refresh`和`flush` `cpu`缓存来实现

* `refresh`是一个读操作，会把内存中的数据读到`cpu`缓存
* `flush`是一个写操作，会把`cpu`缓存中的数据写到内存

#### 3. synchronized、lock与volatile

* `synchronized`：`jvm`会在`MonitorEnter`和`MonitorExit`对应的`cpu`指令中插入内存屏障，从而保证`临界区`代码的`可见性`和`有序性`

> `jvm`会在锁(`synchronized`)获取的地方插入一条`MonitorEnter`指令，而在锁释放的地方插入一条`MonitorExit`指令

* `lock`：`jdk 5`中`lock`的实现是基于`CAS`无锁交换算法的`AQS`(抽象队列同步器)，`CAS`同样也使用内存屏障来保证其可见性

* `volatile`：`jvm`会在`volatile`写操作前后插入一组内存屏障，保证`volatile`变量的写操作相对其之前和之后的代码是有序的，同时也保证了`volatile`变量的可见性

**注意：** 这里的`有序性`是相对而言，即屏障内部的代码相对屏障之前和之后的代码是有序的，但不保证屏障内部的代码之间或者屏障外部的代码之间是有序的

#### 4. 关于可见性与有序性

> 参考[java中的可见性、有序性和原子性]({{ site.url}}/2016/06/28/java-visibility-ordering-atomic/)