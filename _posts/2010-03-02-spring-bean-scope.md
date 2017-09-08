---
layout: post
title: spring中bean scope
categories: [编程, java, spring]
tags: [bean, scope]
---

spring中bean的作用范围：

1. spring context中
* singleton 单例模式，无状态bean
* prototype 多例模式，有状态bean

2. spring mvc中新增的几种范围：
* request 每个request都会创建新的bean实例
* session 会话期共享同一个bean实例
* global session 用于portlet的web应用

> Portlet规范定义了全局Session的概念，它被所有构成某个portlet web应用的各种不同的portlet所共享