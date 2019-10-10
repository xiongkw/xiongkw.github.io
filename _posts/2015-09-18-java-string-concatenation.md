---
layout: post
title: String的+
categories: [编程, java]
tags: [StringBuilder, String]
---

> 上一篇解释创建了几个`String`对象，那`String +` 又是怎么回事呢?

#### 1. 关于StringBuilder的警告

```java
String s = new StringBuilder().append(a).append("b").append("c").toString();
```

> 以上代码在`idea`中会提示警告信息：`'StringBuilder' can be replaced with 'String'`

#### 2. string+规则

通过`idea`提示信息`Replace 'StringBuilder' with 'String'`，重构代码

```
String s = a + "b" + "c";
```

> 结果有点意外，根据以往经验，`String+`应该写成`StringBuilder`，为何`idea`要反其道而行?

#### 3. 查看编译出的class文件

写一段测试代码

```java
        String s1 = a + "b" + "c";
        String s2 = new StringBuilder().append(a).append("b").append("c").toString();
```

```
$ javac Test.java
$ javap -c Test.class
```

查看`class`文件

```
     Code:
       0: new           #2                  // class java/lang/StringBuilder
       3: dup
       4: invokespecial #3                  // Method java/lang/StringBuilder."<init>":()V
       7: aload_0
       8: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      11: ldc           #5                  // String b
      13: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      16: ldc           #6                  // String c
      18: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      21: invokevirtual #7                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      24: astore_1
      25: new           #2                  // class java/lang/StringBuilder
      28: dup
      29: invokespecial #3                  // Method java/lang/StringBuilder."<init>":()V
      32: aload_0
      33: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      36: ldc           #5                  // String b
      38: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      41: ldc           #6                  // String c
      43: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      46: invokevirtual #7                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      49: astore_2
      50: return
```

> 发现两段代码的编译结果完全相同

#### 4. 结论

* `javac`会把`String+`编译成`StringBuilder.append`，所以`String+`不存在执行效率的问题
* 在执行效率相同的情况下，`String+`的代码更简洁