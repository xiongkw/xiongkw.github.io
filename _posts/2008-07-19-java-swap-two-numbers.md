---
layout: post
title: Java交换两个数的值
categories: [编程, java]
tags: [swap]
---


> 很常见的面试笔试题：写一个方法交换两个`int`参数的值，其实是一个陷阱。因为`java`中基本类型传参是传值拷贝，方法内部对形参的交换并不会影响实参

不使用临时变量交换两个数的值：
```java
public static void main(String[] args) {
    int x=3,y=4;
    System.out.println(x+","+y);
    x = x ^ y;
    y = x ^ y;
    x = x ^ y;
    System.out.println(x+","+y);
}
```

注意以下是无效的，因为`java`方法调用中基本类型是值传递：
```java
public void swap(int x, int y){
    //...
}

public static void main(String[] args) {
    int x=3,y=4;
    System.out.println(x+","+y);
    swap(x, y);
    System.out.println(x+","+y);
}
```