---
layout: post
title: Java交换两个数的值
categories: [编程, java]
tags: [java, swap]
---


不使用临时变量交换两个数的值
```java
	public static void swap(int x, int y) {
        System.out.println(x+","+y);
        x = x ^ y;
        y = x ^ y;
        x = x ^ y;
        System.out.println(x+","+y);
    }
```
