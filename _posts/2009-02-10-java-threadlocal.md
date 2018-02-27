---
layout: post
title: Java多线程中的ThreadLocal
categories: [编程, java]
tags: [并发, 多线程]
---


> `J2EE`中`servlet`以单例模式运行在多线程环境中，为了保证线程安全，它必须被设计成无状态的   
> 如果需要在当前线程中保存线程的状态（通俗的说就是定义线程私有的变量，例如每个线程调用该方法的次数），就需要用到`ThreadLocal`了

先看`ThreadLocal`的用法：
```java
public class THreadSafeService {
    //定义一个ThreadLocal变量，用于记录线程调用service的次数
    private ThreadLocal<Integer> counter = new ThreadLocal<Integer>(){
        @Override
        protected Integer initialValue() {
            return new Integer(0);
        }
    };
    
    public void service(){
        //每次调用后counter+1，并set给counter
        Integer c = counter.get();
        counter.set(++ c);
        //do service
    }
    
}

```

`ThreadLocal`怎么保证`counter`变量能被所有线程共享，并且互不干扰呢？看`ThreadLocal`的源码：
```java
    public T get() {
        Thread t = Thread.currentThread();
        //ThreadLocal中维护了一个以当前线程为key的Map?再看getMap实现
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    ThreadLocalMap getMap(Thread t) {
        //Thread类中定义了一个ThreadLocalMap的实例变量
        return t.threadLocals;
    }    

```

回头再看`service`的调用过程
```java
    public void service(){
        //每次调用后counter+1，并set给counter
        Integer c = counter.get();
        // 1. counter.get()方法先获取当前线程
        // 2. 获取当前线程的ThreadLocalMap实例变量
        // 3. 获取ThreadLocalMap中key为this的值
        counter.set(++ c);
        // set调用同get
        
        //do service
    }
```

> 注：`spring`中可以定义`bean`的`scope`为`prototype`来实现有状态的服务