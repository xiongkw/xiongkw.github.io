---
layout: post
title: java中static变量和代码块的初始化
categories: [编程, java]
tags: [static]
---


> `java`中常用`static`表示静态变量，即类变量，可通过类直接访问，而不需要类实例化

#### 先看一个例子
```java
public class StaticTest {

    public static void main(String[] args) {
        System.out.println(InnerClass.class);
    }

    static class InnerClass {
        private static int a = 0;

        static {
            System.out.println("init");
        }

    }
}
```

运行结果：
```
class StaticTest$InnerClass
```

> 为什么`static`块没有执行呢？通常我们认为`static`变量和代码块会在类加载后初始化，而实际上不是这样的

#### 进一步测试

```java
public class StaticTest {

    public static void main(String[] args) {
        System.out.println(InnerClass.class);
        System.out.println(InnerClass.a);
    }

    static class InnerClass {
        private static int a = 0;

        static {
            System.out.println("init");
        }

        private static void method() {
            System.out.println("method");
        }

    }
}
```

运行结果
```
class StaticTest$InnerClass
init
0
```

> 可以看到`static`代码块执行了，可见对`static`属性的访问会引起`static`变量和代码块的初始化。实际上对`static`方法的调用也会起到同样作用


#### 来一道经典面试题
```java
public class StaticTest {

    public static void main(String[] args) {
        System.out.println(InnerClass.class);
        System.out.println(InnerClass.a);
        System.out.println(InnerClass.b);
    }

    static class InnerClass {
        private static InnerClass ic = new InnerClass();
        private static int a = 0;
        private static int b;

        static {
            System.out.println("init");
        }

        private static void method() {
            System.out.println("method");
        }

        public InnerClass() {
            InnerClass.a = 1;
            InnerClass.b = 1;
        }
    }
}
```

运行结果：
```
class StaticTest$InnerClass
init
0
1
```

> 静态变量会按照声明的顺序先依次声明并设置为该类型的默认值，但不赋值为初始化的值，声明完毕后，再按声明的顺序依次设置为初始化的值，如果没有初始化的值就跳过

所以变量`a`的一共被赋值了三次：
```
private static int a; // 声明时为默认值0
new InnerClass(); // 被赋值为1
private static int a = 0; // 初始化时被赋值为0
```

#### 总结

* `static`变量在类加载时仅仅只会被声明，其值为其类型的默认值，例如`int`类型默认值为`0`，'boolean'类型默认值为'false`，`Object`类型默认值为`null`
* `static`变量和代码块在其任意`static`变量或方法第一次被访问时初始化
* 根据以上两点：`static`变量会按照其定义的顺序依次声明且赋默认值，直到其某个`static`变量或方法被访问时，按顺序初始化各`static`变量和代码块
