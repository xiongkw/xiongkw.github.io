---
layout: post
title: mybatis一级缓存
categories: [编程, mybatis]
tags: [cache]
---

> 在一个循环休内查指定的某条数据，结果怎么也查不到

#### 1. 原因

mybatis一级缓存，SqlSession 作用范围，在一个事务中同样的查询只会执行一次

#### 2. 解决办法

* 设置mapper刷新缓存：`flushCache="true"`，建议
* 关闭全局一级缓存：`local-cache-scope=statement`，不建议
* 关掉事务：不太友好
