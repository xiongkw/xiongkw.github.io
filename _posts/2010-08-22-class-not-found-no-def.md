---
layout: post
title: java中的ClassNotFoundException和NoClassDefFoundError
categories: [编程, java]
tags: [ClassLoader]
---


> `ClassNotFoundException`和`NoClassDefFoundError`是两个比较常见的异常，基本意思都是找不到`class`，但是二者到底有什么区别呢

#### 1. 一个例子
```java
public static void main(String[] args) {
    try {
        Class.forName("org.springframework.beans.BeanUtils");
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }

    try {
        Class<BeanUtils> beanUtilsClass = BeanUtils.class;
        System.out.println(beanUtilsClass);
    } catch (Exception e) {
        e.printStackTrace();
    }
    
}
```

先编译，然后使用命令运行`java ClassLoaderTest`

```
java.lang.ClassNotFoundException: org.springframework.beans.BeanUtils
        at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        at java.lang.Class.forName0(Native Method)
        at java.lang.Class.forName(Class.java:264)
        at ClassLoaderTest.main(ClassLoaderTest.java:8)
Exception in thread "main" java.lang.NoClassDefFoundError: org/springframework/beans/BeanUtils
        at ClassLoaderTest.main(ClassLoaderTest.java:14)
Caused by: java.lang.ClassNotFoundException: org.springframework.beans.BeanUtils
        at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        ... 1 more
```

#### 2. 看文档

`ClassNotFoundException`

```
Thrown when an application tries to load in a class through its string name using:
The forName method in class Class.
The findSystemClass method in class ClassLoader .
The loadClass method in class ClassLoader.
but no definition for the class with the specified name could be found.
```

`NoClassDefFoundError`

```
Thrown if the Java Virtual Machine or a ClassLoader instance tries to load in the definition of a class (as part of a normal method call or as part of creating a new instance using the new expression) and no definition of the class could be found.   
The searched-for class definition existed when the currently executing class was compiled, but the definition can no longer be found.
```

#### 3. 总结

* `ClassNotFoundException`一般发生在调用`Class.forName`、`ClassLoader.findSystemClass`和`ClassLoader.loadClass`时，原因是在类路径下找不到传入的`className`，可能是`className`错误，也可能是缺少了`jar`包
* `NoClassDefFoundError`一般是编译期通过(例如`new MyClass()`)，但运行时找不到类，原因是运行时类路径和编译时不同
* 二者一个`Exception`，一个`Error`也说明了前者多半是因为参数错误，是可捕获处理的，而后者多是运行时类路径问题