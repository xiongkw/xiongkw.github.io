---
layout: post
title: java内存溢出分析之-栈溢出
categories: [编程, java]
tags: [stack, oom, jvisualvm]
---

> `stack`(`jvm`栈)，用于存放线程中方法的调用信息，包括局部变量、返回值、方法调用链等

#### 1. 关于Stack Overflow

除了是`java`编码中令人讨厌的异常之一，`Stack Overflow` 还是大名鼎鼎的`IT`技术网站

> Stack Overflow is the largest online community for programmers to learn, share their knowledge, and advance their careers

#### 2. stack内存

`stack`内存设置
```
-Xss128k //默认1M
```

#### 3. 一个例子

本例通过方法无穷递归让线程栈溢出
```java
public class StackOverFlow {
    private int i;
    public void plus() throws InterruptedException {
        i++;
        if(i % 1000==0){
            Thread.sleep(3000);
        }
        plus();
    }

    public static void main(String[] args) throws InterruptedException {
        StackOverFlow stackOverFlow = new StackOverFlow();
        try {
            stackOverFlow.plus();
        } catch (Error e) {
            System.out.println(stackOverFlow.i);
            e.printStackTrace();
        }
    }
}
```

#### 4. 参考

* [java内存模型]({{site.url}}/2017/06/27/java-memory/)
* [java中的GC]({{site.url}}/2017/07/01/java-gc/)