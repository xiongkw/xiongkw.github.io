---
layout: post
title: Java中的四种引用类型
categories: [编程, java]
tags: [reference]
---

> java中的引用类型有四种

#### 1. 强引用
只要引用存在，垃圾回收器永远不会回收
```java
Object obj = new Object();   

//对象使用完之后赋null是一个良好的编码习惯
obj = null;
  
```


#### 2. 软引用
内存溢出之前才会被回收，常用于缓存
```java
Object obj = new Object();   
SoftReference<Object> sf = new SoftReference<Object>(obj);   
sf.get();//有时候会返回null   
```

#### 3. 弱引用
下一次垃圾回收时回收   
```java
Object obj = new Object();   
WeakReference<Object> wf = new WeakReference<Object>(obj);
obj = null;
wf.get();//有时候会返回null
wf.isEnQueued();//返回是否被垃圾回收器标记为即将回收的垃圾
```
弱引用是在第二次垃圾回收时回收，短时间内通过弱引用取对应的数据，可以取到，当执行过第二次垃圾回收时，将返回null。
弱引用主要用于监控对象是否已经被垃圾回收器标记为即将回收的垃圾，可以通过弱引用的isEnQueued方法返回对象是否被垃圾回收器标记。

#### 4. 虚引用  
垃圾回收时回收，无法通过引用取到对象值
```java
Object obj = new Object();
PhantomReference<Object> pf = new PhantomReference<Object>(obj);
obj=null;
pf.get();//永远返回null
pf.isEnQueued();//返回是否从内存中已经删除
```
虚引用是每次垃圾回收的时候都会被回收，通过虚引用的get方法永远获取到的数据为null，因此也被成为幽灵引用。
虚引用主要用于检测对象是否已经从内存中删除。