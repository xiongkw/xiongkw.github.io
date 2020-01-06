---
layout: post
title: JDK8中的Stream
categories: [编程, linux, java]
tags: [jdk8, parallelStream]
---

> 最近看到项目中有很多`Collection.parallelStream().forEach()`的写法，其意是通过并行方式查询数据库，以提高程序运行效率

#### 1. Stream

查看`Collection.stream()`和`Collection.parallelStream()`，发现其返回的是一个`Stream`对象

#### 2. 串行和并行Stream

查看`forEach`源码：
```
public void forEach(Consumer<? super E_OUT> action) {
    if (!isParallel()) {
        sourceStageSpliterator().forEachRemaining(action);
    }
    else {
        super.forEach(action);
    }
}
```

```java
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
    assert getOutputShape() == terminalOp.inputShape();
    if (linkedOrConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
    linkedOrConsumed = true;

    return isParallel()
           ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
           : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
}
```

```
public <S> Void evaluateParallel(PipelineHelper<T> helper,
                                 Spliterator<S> spliterator) {
    if (ordered)
        new ForEachOrderedTask<>(helper, spliterator, this).invoke();
    else
        new ForEachTask<>(helper, spliterator, helper.wrapSink(this)).invoke();
    return null;
}
```

> 发现并行执行的底层用的是`JDK7`中的`Fork/Join`框架，其实就是多线程并发执行

#### 3. 问题

回到问题的出发点来看：循环遍历一个数组，执行数据库查询，并行肯定比串行的用时少(不考虑极端情况)

> 使用并行`Stream`看似提高了效率，其实是入了坑，因为问题的本质不在数据库操作方式的并行和串行，而是在于数据库操作次数的多少

#### 4. 解决方案

使用批量操作，一次查出所有需要的数据，再循环赋给数组的每个元素
