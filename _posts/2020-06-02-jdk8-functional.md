---
layout: post
title: JDK8中的函数式编程
categories: [编程, java]
tags: []
---


> `JDK14`早就发布了，而我才刚切入`JDK8`，中年大叔果然是要被时代淘汰的

#### 1. 函数式编程

##### 1.1 Lambda表达式

常规写法：
```java
new Thread(new Runnable(){
    public void run(){
        System.out.println("Haha");
    }
}).start();
```

Lambda写法：
```java
new Thread(() -> {
    System.out.println("Haha");
}).start();
```

##### 1.2 FunctionalInterface注解

标注在接口上，表示该接口是个`函数式接口`，可以用于Lambda表达式(当然不标注也能用，前提是接口有且仅有一个抽象方法)

##### 1.3 常用函数式接口

* `Function`：把一个对象转换为另一个对象，有入参，有返回

```java
public interface Function<T, R> {
   R apply(T t);
}
```

* `Consumer`：消费一个对象，无返回

```java
public interface Consumer<T> {
   void accept(T t);
}
```

* `Supplier`：返回一个对象，无入参

```java
public interface Supplier<T> {
   T get();
}
```

* `Predicate`：判断条件是否成立，有入参

```java
public interface Predicate<T> {
   boolean test(T t);
}
```

##### 1.4 方法引用

形如：obj::method，用于简写Lambda表达式

```java
list.stream().map(obj::method).collect(Collectors.toList());
```

#### 2. Optional

##### 2.1 空值判断

```java
Optional.offNullable(obj).ifPresent(o->{

});
```

##### 2.2 常见用法

```java
Optional.offNullable(obj).map(o->{ // 对象转换
    return o;
}).orElseGet(() -> { // Supplier 接口

});
```

#### 3. Stream

##### 3.1 Stream的创建

```java
Stream.of("a","b");

list.stream();
```

##### 3.2 Stream基本用法

```java
list.stream().map(o->{ // 对象转换
    return o;
}).peek(o->{ // 消费

}).filter(Obj::isValid) // 过滤
.distinct() // 去重（需要实现equals和hashCode方法）
.sort(Comparator.comparing(Obj::getId)) // 排序
.collect(Collecotrs.toList()) // 收集到集合
```

##### 3.3 Stream的统计方法

```java
Optional<Integer> maxOpt = list.stream().max(Comparator.comparing(o->o));
Optional<Integer> minOpt = list.stream().min(Comparator.comparing(Function.identity()));
long count = list.stream().count();
```

##### 3.4 Stream的分组方法

```java
HashMap<String, List<Obj>> collect = list.stream()
    .collect(Collectors.groupingBy(Obj::getType, HashMap::new, Collectors.toList()));
```

##### 3.5 并行Stream

```java
list.parallelStream().forEach(o->{

});
```

##### 3.6 Stream平铺

```java
List<Obj[]> list = ...;
list.stream().flatMap(Stream::of).collect(Collectors.toList());
```

#### 4. 参考

* [函数式编程初探](https://www.ruanyifeng.com/blog/2012/04/functional_programming.html)
* [JDK8中Stream使用解析](https://www.cnblogs.com/pi-laoban/p/14855018.html)
