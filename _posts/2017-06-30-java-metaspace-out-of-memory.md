---
layout: post
title: java内存溢出分析之-元空间溢出
categories: [编程, java]
tags: [metaspace, oom, jvisualvm]
---

> `Metaspace`(元空间)，是`jdk8`的概念，用于存放类的元信息，`jdk7`之前是属于`Perm gen`(永久区)

#### 1. 一个例子

本例通过不断生成并加载新的类让元空间溢出
```java
public class MetaspaceOutOfMenory {
    /*
     * vm 参数 -XX:MaxMetaspaceSize=10M
     */
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        for (int i = 0; ; i++) {
            CtClass cl = pool.makeClass("MyClass" + i);
            cl.toClass();
            if (i % 1000 == 0) {
                Thread.sleep(3000);
            }
        }
    }
}
```

在`jvisualvm`中可以看到元空间内存不断增长，类数量也不断增加

![]({{site.url}}/public/images/2017-06-30-java-metaspace-out-of-memory-1.png)

通过`jvisualvm`中两个堆`dump`的比较，看到新装入的类`MyClass*`
![]({{site.url}}/public/images/2017-06-30-java-metaspace-out-of-memory-2.png)

异常信息
```
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
	at javassist.ClassPool.toClass(ClassPool.java:1085)
	at javassist.ClassPool.toClass(ClassPool.java:1028)
	at javassist.ClassPool.toClass(ClassPool.java:986)
	at javassist.CtClass.toClass(CtClass.java:1079)
	at com.mywork.mem.MetaspaceOutOfMenory.main(MetaspaceOutOfMenory.java:18)
```

#### 2. 参考

* [java内存模型]({{site.url}}/2017/06/27/java-memory/)
* [java中的GC]({{site.url}}/2017/07/01/java-gc/)