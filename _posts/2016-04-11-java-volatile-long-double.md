---
layout: post
title: java中long/double的非原子性和volatile
categories: [编程, java]
tags: [long, double, 原子性, volatile]
---

#### 1. 参考

> For the purposes of the Java programming language memory model, a single write to a non-volatile long or double value is treated as two separate writes: one to each 32-bit half. This can result in a situation where a thread sees the first 32 bits of a 64-bit value from one write, and the second 32 bits from another write.      
> Writes and reads of volatile long and double values are always atomic.   
> Writes to and reads of references are always atomic, regardless of whether they are implemented as 32-bit or 64-bit values.      
> Some implementations may find it convenient to divide a single write action on a 64-bit long or double value into two write actions on adjacent 32-bit values. For efficiency's sake, this behavior is implementation-specific; an implementation of the Java Virtual Machine is free to perform writes to long and double values atomically or in two parts.      
> Implementations of the Java Virtual Machine are encouraged to avoid splitting 64-bit values where possible. Programmers are encouraged to declare shared 64-bit values as volatile or synchronize their programs correctly to avoid possible complications.

> 参考[Non-atomic Treatment of double and long](https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.7)

#### 2. 关于long和double

`long`和`double`长度都是`8`字节，也就是`64bits`。在`32`位操作系统上对`64`位的数据的读写要分两步完成，每一步取`32`位数据

#### 3. 多线程下的long和double

由于编译器的重排序机制，在`32`位`jvm`上写一个`long`变量将会是两个无序的操作
```
write(前32位);
write(后32位);
```

当两个线程同时写时，可能发生的是：
```
线程A -> write(前32位)
线程B -> write(后32位)

线程B -> write(前32位)
线程A -> write(后32位)
```

最终结果是前`32`位是线程`B`写的，后`32`位是线程`A`写的

#### 4. volatile

`volatile`本身不保证获取和设置操作的原子性，仅仅保持修改的可见性。但是`jvm`保证声明为`volatile`的`long`和`double`变量的读写操作都是原子的
