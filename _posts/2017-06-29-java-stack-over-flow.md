---
layout: post
title: java内存溢出分析之-栈溢出
categories: [编程, java]
tags: [stack, oom, jvisualvm]
---

> stack(jvm栈)，用于存放线程中方法的调用信息，包括局部变量、返回值、方法调用链等

除了是java编码中令人讨厌的异常之一，`Stack Overflow` 还是大名鼎鼎的IT技术网站
> Stack Overflow is the largest online community for programmers to learn, share their knowledge, and advance their careers

stack内存设置
```
-Xss128k //默认1M
```
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