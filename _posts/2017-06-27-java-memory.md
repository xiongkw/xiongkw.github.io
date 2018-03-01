---
layout: post
title: java内存模型
categories: [编程, java]
tags: [jvm, 内存, GC]
---

> 本文基于`java 8`

#### 1. jvm内存划分

![]({{site.url}}/public/images/2017-06-27-java-memory.png)

#### 2. stack
`java栈`用于存储线程范围的方法调用数据，其`方法调用栈`中的每个`方法调用`分别存储在不同的`栈桢`中

> `-Xss`: 指定线程的栈大小，默认`1M`

#### 3. PC寄存器

多线程技术基于`CPU`的分时调度，所以在线程上下文切换时，必须要保存线程当前状态(执行到哪条指令了)，以便下次接着运行，`PC寄存器`便是用于存储线程运行状态，`PC寄存器`是线程私有的

#### 4. Native Method Stack

`本地方法栈`: 用于支持`native`方法(`c`编写的方法)的执行，作用和`java 栈`类似

#### 5. Heap

`java堆`用于存放实例对象，是`jvm`中内存占用最大的部分，是垃圾回收器主要管理的地方，在`java8`中元空间也划到了`堆`中，所以`堆`包括`新生代`、`老年代`和元空间

> `-Xms`: 指定堆的初始内存大小，默认为物理内存的`1/64`   
> `-Xmx`: 指定堆的最大内存，默认为物理内存的`1/4`

#### 5.1 新生代和老年代

`jvm`通过划分`新生代`和`老年代`以更精细的划分`GC`任务，通常情况下`new`创建的对象放于`新生代`，而当指定次数(默认`15`次)的`Young GC`发生后，如果对象依然存活，则会被转移到`老年代`

关于`Young GC`参考[java中的GC]({{site.url}}/2017/07/01/java-gc/)

> `-Xmn`: 指定新生代内存大小   
> `-XX:NewSize`: 新生代内存初始值，和`-Xmn`等价
> `-XX:MaxNewSize`: 新生代内存最大值   
> `-XX:NewRatio`: 新生代与老年代的比值，`-XX:NewRation=4`表示新生代和老年代各占`1/5`和`4/5`，当设置`-Xmn`后此项设置无效

#### 5.2 Eden和Survivor

`新生代`又划分为`Eden`、`From Survivor`和`To Survivor`

* `Eden`: 新创建的对象会存放在`Eden`,经过一次`Young GC`后`Eden`中仍存活的对象会转移到`To Survivor`
* `From Survivor`: 经过一次`Young GC`后`From Survivor`中的对象会转移到`To Survivor`
* `To Survivor`: `Young GC`执行之前`To Survivor`是空的，执行过后`From Survivor`和`To Survivor`会互换

> `-XX:SurvivorRatio`: `Eden`与`Survivor`的比值，默认为`8`，表示`Eden`占`8/10`，两个`Survivor`各占`1/10`

#### 6. 元空间

`Metaspace`: 用于存放类信息等，`java7`之前称为`永久代`(`Permanant Generation`)

> `-XX:MetaspaceSize`: 元空间初始内存大小
> `-XX:MaxMetaspaceSize`: 元空间最大内存

#### 7. 永久代

>`perm gen`(永久代)是`java7`以前的概念，用于存放`类信息`、`常量`、`静态变量`等，从`jdk7`开始，常量和静态变量已经转移到`heap`中了