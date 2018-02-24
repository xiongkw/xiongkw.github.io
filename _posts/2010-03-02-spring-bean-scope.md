---
layout: post
title: spring中bean scope
categories: [编程, java, spring]
tags: [bean, scope]
---


> `spring`中不同类型的`bean`可以有不同的生命周期

#### 1. spring context中
* `singleton`: 单例模式，最常用的模式，无状态`bean`
* `prototype`: 多例模式，常用于有状态`bean`

#### 2. spring mvc中
* `request`: 每个`request`都会创建新的`bean`实例
* `session`: 会话期共享同一个`bean`实例
* `global session`: 用于`portlet`的`web`应用

> `Portlet`规范定义了全局`Session`的概念，它被所有构成某个`portlet web`应用的各种不同的`portlet`所共享