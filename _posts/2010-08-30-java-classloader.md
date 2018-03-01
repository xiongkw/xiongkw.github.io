---
layout: post
title: java中的ClassLoader
categories: [编程, java]
tags: [ClassLoader]
---


#### 1. 概念

`ClassLoader`: 用来加载`class`文件到`jvm`中并生成一个`Class`对象实例，除此以外还可以用来加载一些资源文件，如图像文件和配置文件等

`java`中的`ClassLoader`可分为三类：

* `Bootstrap ClassLoader`: 启动类加载器，由`C++`编写，用于加载`JAVA_HOME/lib`中的所有类
* `ExtClassLoader`: `URLClassLoader`的子类，用于加载`JAVA_HOME/lib/ext`中的类
* `AppClassLoader`: `URLClassLoader`的子类，用于加载`classpath`中的类


#### 2. 一个例子
```java
public static void main(String[] args) {
    ClassLoader cl = ClassLoaderTest.class.getClassLoader();
    while (cl != null) {
        System.out.println(cl.getClass().getName());
        cl = cl.getParent();
    }
}
```

运行结果：

```
sun.misc.Launcher$AppClassLoader
sun.misc.Launcher$ExtClassLoader
```

> 可以看到运行结果里没有`Bootstrap ClassLoader`，原因是`AppClassLoader`的`parent`得到的是`null`，可见`Bootstrap ClassLoader`是一个比较特殊的`ClassLoader`，其由`C++`编写并嵌套在`jvm`内部

#### 3. 关于双亲委派

我们知道，同一个类被不同的`ClassLoader`加载所生成的`Class`是不相同的，为了保证同一个类只能被加载一次，`ClassLoader`采用双亲委派模式，即优先由其父`ClassLoader`加载，只有父`ClassLoader`加载不到时才由自己加载

> 虽然如此，但是我们也可以通过继承`ClassLoader`实现自定义加载器，从而改写`双亲委派`的逻辑