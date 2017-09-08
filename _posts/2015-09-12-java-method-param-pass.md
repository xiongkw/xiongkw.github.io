---
layout: post
title: 从jvm内存角度看java方法参数的传递
categories: [编程, java]
tags: [jvm, stack, heap, 传参]
---

> 引用`Jvm规范` Like the Java programming language, the Java Virtual Machine operates on two kinds of types: primitive types and reference types. There are, correspondingly, two kinds of values that can be stored in variables, passed as arguments, returned by methods, and operated upon: primitive values and reference values.   
> The Java Virtual Machine contains explicit support for objects. An object is either a dynamically allocated class instance or an array. A reference to an object is considered to have Java Virtual Machine type reference. Values of type reference can be thought of as pointers to objects. More than one reference to an object may exist. Objects are always operated on, passed, and tested via values of type reference.

* `jvm` 操作的对象有两种：基本类型和引用类型，其值分别对应基本类型值(boolean, byte, char, short, int, float, long, double)和引用类型值(对象在堆中的地址，就像`C语言`中的`指针`)   
* 对象的操作、传递和检查都是通过其引用(reference)的值来进行，这里已经确定对象的传参方式是传值，即传其引用的值。

以下通过内存分配来分析

声明一个基本类型变量和一个对象变量时的内存分配:
```java
public void test(){
    //定义一个int变量，其值保存在栈帧中
    int a = 1;
    // 基本类型赋值直接拷贝其值
    int b = a;//b == 1
    //再次给a赋值并不会改变b
    a = 2;//b==1;
    //通过关键字new在堆中创建了一个String对象，其引用s的值保存在栈帧中
    String s = new String("abc");
    //对象赋值拷贝其引用值，引用副本指向的是对象abc
    String c = s;//c == abc
    //c是s的副本，所以对c的再次赋值不会改变s
    String c = new String("xx");//c == xx
    c = s;// c == abc
    // 对s的再次赋值只是改变了其在栈中的引用值，不会影响到c
    s = "def";//c == abc
}
        
```
> 关于赋值操作`=`   
> 1. 基本类型赋值是值拷贝   
> 2. 对象赋值是引用值的拷贝   
> 3. 既然是拷贝，则不会影响到原值或者引用值

基本类型参数
```java
public static void test1(int i){
    System.out.println(i);
    //基本类型的赋值，直接改变其栈帧中的内存值，所以不会影响外部变量
    i = 1;
}

public static void main(String[] args){
    int a = 0;
    //参数传递本质上就是用实参对形参进行赋值操作，这里基本类型是值拷贝
    test1(a);
    System.out.println(a);// 0
}
```

对象参数
```java
public static void test(StringBuilder sb){
    System.out.println(sb);//abc
    //这里是对象引用值的拷贝，指向的是堆中同一个对象，所以可能通过其调用堆中对象的方法
    sb.append("d");
    //sb只是原对象引用值的拷贝，再次赋值不会改变原对象的引用值
    sb = new StringBuilder("xx");
}

public static void main(String[] args){
    StringBuilder sb = new StringBuilder("abc");
    //对象的参数传递也是赋值操作，是对象引用值的拷贝
    test(sb);
    System.out.println(sb);//abcd
}
```

> 总结：java中的一切参数传递都是值传递，所不同的是`基本类型`是传基本类型值，`对象`是传引用的值，不论是传值或是传引用值，其本质都是通过赋值操作在`栈帧`中创建一个副本。   
> 另外：`jvm`规定`pritive`、`reference`和`returnAddress`三种类型的值保存在`栈`中，而对象和数组保存在`堆`中

java中是没有`传引用`的，关于`传引用`，参见[C语言传参中的传值和传引用]({{site.url}}/2015/09/10/c-method-param-pass)