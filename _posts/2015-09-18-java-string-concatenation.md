---
layout: post
title: String的+和intern
categories: [编程, java]
tags: [jvm, heap, String]
---

> 上一篇解释创建了几个`String`对象，那`String +` 又是怎么回事呢?

WARN: 下面代码比较多，可能会看的人头晕
```java
    //1. "12"在编译期已经确定会放入常量池
    String s = new String("12");
    // 所以这里返回的是常量池中"12"的引用(附2)
    String i1 = s.intern();
    System.out.println(s == i1);//false
    // = "12"会从常量池中检索，如果常量池不存在，则放入常量池，返回引用
    String s2 = "12";
    System.out.println(s2 == i1);//true
    
    //2. String常量+对象的结果并不会放入常量池(附1)
    String s3 = new String("3") + "4";
    // 因为"34"不在常量池中，所以intern操作会把s3放入常量池，并返回其引用，这个引用和s3是相同的(附2)
    String i3 = s3.intern();
    System.out.println(s3 == i3);//true
    
    //3. 这里会产生三个String常量:"5"、"6"和"56"，并且s5直接引用常量"56"(附1)
    String s5 = "5" + "6";
    // s5本身即在常量池中，所以这里返回的和s5相同(附2)
    String i5 = s5.intern();
    System.out.println(s5 == i5);//true
    
    //4.
    String s7 = "7";
    // s7是变量，所以"8"+s7不是常量表达式，其结果不会放入常量池(附1)
    String s8 = "8" + s7;
    // 同2.
    String i8 = s8.intern();
    System.out.println(s8 == i8);//true
    
    //5. "10"会放入常量池
    String s10 = "10";
    String s11 = "0";
    // s12不会放入常量池(附1)
    String s12 = "1" + s11;
    System.out.println(s10 == s12);//false
    //"10"已经存在于常量池中，所以这里直接返回了s10的引用(附2)
    String i12 = s12.intern();
    System.out.println(s10 == i12);//true
    System.out.println(s12 == i12);//false
```

*附1:*
 
> The string concatenation operator + implicitly creates a new String object when the result is not a constant expression .   
> 引用[Java Language Specification](http://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.3.3)


*附2:*

```java
/**
     * Returns a canonical representation for the string object.
     * <p>
     * A pool of strings, initially empty, is maintained privately by the
     * class {@code String}.
     * <p>
     * When the intern method is invoked, if the pool already contains a
     * string equal to this {@code String} object as determined by
     * the {@link #equals(Object)} method, then the string from the pool is
     * returned. Otherwise, this {@code String} object is added to the
     * pool and a reference to this {@code String} object is returned.
     * <p>
     * It follows that for any two strings {@code s} and {@code t},
     * {@code s.intern() == t.intern()} is {@code true}
     * if and only if {@code s.equals(t)} is {@code true}.
     * <p>
     * All literal strings and string-valued constant expressions are
     * interned. String literals are defined in section 3.10.5 of the
     * <cite>The Java&trade; Language Specification</cite>.
     *
     * @return  a string that has the same contents as this string, but is
     *          guaranteed to be from a pool of unique strings.
     */
    public native String intern();
```

> When the intern method is invoked, if the pool already contains a string equal to this String object as determined by the equals(Object) method, then the string from the pool is returned. Otherwise, this String object is added to the pool and a reference to this String object is returned.

解释一下:
```java
    // 根据附1. 变量加常量是非常量表态式，所以"34"不会放到常量池
    String s3 = new String("3") + "4";
    // 根据附2. 不在常量池中的string，调用intern方法会把值放入常量池，并返回该对象的引用
    String i3 = s3.intern();
    // 所以s3.intern()返回的引用就是s3
    System.out.println(s3 == i3);//true
```

