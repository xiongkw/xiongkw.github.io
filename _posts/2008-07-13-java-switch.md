---
layout: post
title: Java中的switch语句
categories: [编程, java]
tags: [switch]
---


> `java`中`switch`语句按`case`顺序匹配，在碰到第一个匹配的`case`后，会按顺序执行以后的`case`代码，直到碰到`break`或`return`

一段示例代码：
```java
    public static void main(String[] args) {
        switch (1) {
        case 0:
            System.out.println(0);
        case 1:
            System.out.println(1);
        case 2:
            System.out.println(2);
            break;
        case 3:
            System.out.println(3);
        }
    }
```

结果

```
1
2
```
