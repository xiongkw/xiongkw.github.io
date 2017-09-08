---
layout: post
title: jvm内存划分
categories: [编程, java]
tags: [jvm, 内存]
---

> 本文基于jdk8

JVM内存主要分为如下部分

* heap   
堆内存，用于存放实例对象，是jvm中内存占用最大的部分，是垃圾回收器主要管理的地方。

* stack   
这里说的是线程栈，线程中方法调用链都存在这里，具体包括局部变量、返回值、方法调用链等。

* metaspace   
元空间，用于存放类信息

> perm gen(永久代)是jdk7以前的概念，用于存放类信息、常量、静态变量，从jdk7开始，常量和静态变量已经转移到heap中了