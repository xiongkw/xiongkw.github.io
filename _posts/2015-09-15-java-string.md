---
layout: post
title: 创建了几个String对象
categories: [编程, java]
tags: [jvm, heap, String]
---

> 一个经典的`java`面试题

#### 1. 问题

以下代码创建了几个`String`对象?
```java
String str = new String("abc");
```

#### 2. String的特点

* `String`的内容不可变
```java
/** The value is used for character storage. */
    private final char value[];
```
> String 的内容用一个声明为`final`的`char`数组存放，`final`决定了它的内容不可变

* `String`常量池
`java`中的对象无非是由`基本类型`和`String`组成，夸张的说java中占用内存最多的对象(`基本类型`不是对象，并且存储在`栈`中)归根结底都是`String`。   
那么`String`的实例能有多少呢？光是26个英文字母的排列组合(26^26)已经是一个天文数字了。为了节省存储空间，`jvm`对`String`使用了不严格的单例模式(一个`String`实例只存储一份)，这就是`String常量池`。

```java
String s1 = "abc";
String s2 = "abc";
System.out.println(s1 == s2);//true
```
> 以上代码说明`abc`的实例只保存了一份

* `String`常量的定义
```java
String s1 = "abc";
```
> 使用`""`定义的`String`会被`编译器`认为是`常量`，从而会被放入`String常量池`，注意这个动作发生在`编译期`

* 运行时对`String`常量的操作
```java
String s = new String("a") +"bc";
s.intern();
```

> `String`对象提供了一个`native`方法`intern`，用于在运行时把`String`实例放到`String`常量池中，并返回其引用，所以以下推断永远成立   
> `s1.equals(s2) => s1.intern() == s2.intern()`

#### 3. 创建了几个String

现在再看以下代码创建了几个`String`对象
```java
String str = new String("abc");
```

答案是两个：
* 一个是`"abc"`，在`编译期`被当作常量，`运行期`存放在`常量池`中
* 一个是`str`，在`运行期`通过`new`创建，存放在`堆`中

实际上`new String("abc")`并没有生成新的`char[]`，看源码：
```java
    public String(String original) {
    //这是仅仅是把原String的value(char[])赋值给了this.value，并没有创建新的char
        this.value = original.value;
        this.hash = original.hash;
    }
```