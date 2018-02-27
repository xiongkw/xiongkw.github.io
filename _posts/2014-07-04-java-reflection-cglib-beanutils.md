---
layout: post
title: Java中使用cglib、beanutils和反射的比较
categories: [编程, java]
tags: [反射, cglib, beanutils]
---


> `Java`中通过反射调用对象的方法有很多，常见的`Java reflection api`,`cglib`,`commons-beanutils`,`commons-lang`等

以下测试基于`jdk8`

#### 1. 定义一个类

```java
public class User {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

}
```

#### 2. 直接调用对象方法
```java
public static void main(String[] args) {
    long start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        new User();
    }
    long current = System.currentTimeMillis();
    System.out.println("New 1000000 instance time: " + (current - start));
    User user = new User();
    
    start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        user.setName("aaa");
    }
    current = System.currentTimeMillis();
    System.out.println("Call setName 1000000 times: " + (current - start));
    
    start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        user.getName();
    }
    current = System.currentTimeMillis();
    System.out.println("Call getName 1000000 times: " + (current - start));

}
```

运行结果：
```
New 1000000 instance time: 8
Call setName 1000000 times: 1
Call getName 1000000 times: 0
```

#### 3. 使用`Java Reflection Api`

```java
public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("User");
    long start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        clazz.newInstance();
    }
    long current = System.currentTimeMillis();
    System.out.println("new instance 10000 times: "+(current-start));

    Method set = clazz.getDeclaredMethod("setName", String.class);
    Object user = clazz.newInstance();
    start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        set.invoke(user, "aaa");
    }
    current = System.currentTimeMillis();
    System.out.println("set method 1000000 times: "+(current-start));

    Method get = clazz.getDeclaredMethod("getName", new Class[0]);
    start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        get.invoke(user);
    }
    current = System.currentTimeMillis();
    System.out.println("get method 1000000 times: "+(current-start));
}
```

运行结果：
```
new instance 10000 times: 90
Invoke setName 1000000 times: 50
Invoke getName 1000000 times: 48
```

#### 4. 使用`commons-lang`
```java
public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, Exception {
    User user = new User();
    long start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        MethodUtils.invokeMethod(user, "setName", "aaa");
    }
    long current = System.currentTimeMillis();
    System.out.println("set method 1000000 times: "+(current-start));

    start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        MethodUtils.invokeMethod(user, "getName", null);
    }
    current = System.currentTimeMillis();
    System.out.println("get method 1000000 times: "+(current-start));
}
```

运行结果：
```
Invoke setName 1000000 times: 566
Invoke getName 1000000 times: 555
```

#### 5. 使用`commons-beanutils`
```java
public static void main(String[] args) throws IllegalAccessException, Exception {
     User user = new User();
     long start = System.currentTimeMillis();
     for (int i = 0; i < 1000000; i++) {
         BeanUtils.setProperty(user, "name", "aaa");
     }
     long current = System.currentTimeMillis();
     System.out.println("set method 1000000 times: " + (current - start));
 
     start = System.currentTimeMillis();
     for (int i = 0; i < 1000000; i++) {
         BeanUtils.getProperty(user, "name");
     }
     current = System.currentTimeMillis();
     System.out.println("Get method 1000000 times: " + (current - start));
 }
```

运行结果：
```
Invoke setName 1000000 times: 1060
Invoke getName 1000000 times: 575
```

#### 6. 使用`cglib`
```java
public static void main(String[] args) throws Exception {
    FastClass fc = FastClass.create(User.class);
    FastMethod set = fc.getMethod("setName", new Class[] { String.class });
    
    User user = new User();
    long start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        set.invoke(user, new String[]{"aaa"});
    }
    long current = System.currentTimeMillis();
    System.out.println("set method 1000000 times: "+(current-start));
    FastMethod get = fc.getMethod("getName", new Class[0]);
    start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
        get.invoke(user, null);
    }
    current = System.currentTimeMillis();
    System.out.println("get method 1000000 times: "+(current-start));
}
```

运行结果：
```
Invoke setName 1000000 times: 10
Invoke getName 1000000 times: 3
```

#### 7. 总结

* 直接调用对象方法要比通过反射调用快得多，所以如果要追求高性能，则慎用反射
* 读操作比写更快
* 通过反射调用，最快的是`cglib`，其次是`Java Reflection api`，`commons-lang`和`commons-beanutils`最差