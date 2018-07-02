---
layout: post
title: Storm中的定时器
categories: [编程, java]
tags: [storm, tick]
---


> `Java`中我们常用`Timer`或`ExecutorService`来实现定时任务调度

#### 1. 简介

> `Storm`中内置了一种定时机制——`tick`，它能够让`bolt`的所有`task`每隔一段时间收到一个来自`systemd`的`tick stream`的`tick tuple`，`bolt`收到这样的`tuple`后可以根据业务需求完成相应的处理

#### 2. 设置Tick定时器

```java
public class MyBolt extends BaseRichBolt {

    @Override
    public Map<String, Object> getComponentConfiguration() {
        Config config = new Config();
        config.put(Config.TOPOLOGY_TICK_TUPLE_FREQ_SECS, 10);
        return config;
    }

}
```

> 继承`BaseRichBolt`并重写`getComponentConfiguration`方法，设置定时器

#### 3. 处理定时任务

```java

    public void execute(Tuple input) {
        if (TupleUtils.isTick(input)) {
            doTask();
            return;
        }

        // doExecute

    }
```

> `if (TupleUtils.isTick(input))`: 收到定时器发出的`tuple`后执行定时任务