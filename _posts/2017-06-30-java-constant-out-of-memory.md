---
layout: post
title: java内存溢出分析之-常量池溢出
categories: [编程, java]
tags: [constant, oom, jvisualvm]
---

> `jdk7`开始，常量池从永久代转移到了堆内存中

#### 一个例子

本例通过不断写入常量池让永久代溢出(`jdk7`之后是堆溢出)
```java
public class ConstantOutOfMemory {
    public static void main(String[] args) throws InterruptedException {
        List<String> list = new ArrayList<>();
        long i = 0;
        while (true) {
            list.add(String.valueOf(i++).intern());
            if (i % 100000 == 0) {
                System.out.println(i);
                Thread.sleep(3000);
            }
        }
    }
}
```

* `jdk6`中` -XX:MaxPermSize=10M` 异常：`perm gen out of memory`
* `jdk7`中 `-Xms32M -Xmx32M` 异常：`heap space out of memory`