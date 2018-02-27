---
layout: post
title: Java中的Fork/Join
categories: [编程, java]
tags: [fork, join]
---


> 在`IT`世界，对大任务`分解/合并`的思想最早应该是在`Google`的`MapReduce`论文中提出，后来`jdk7`也发布了类似的框架(`Fork/Join`)

#### 1. MapReduce
`MapReduce`最早是由`Google`公司研究提出的一种面向大规模数据处理的并行计算模型和方法，`Hadoop`便是使用其作为分布式计算框架

> 其核心思想就把一个巨大的任务分解成多个小任务，每个小任务都可以分散到具体的执行机器，最后再合并小任务的结果

#### 2. Fork/Join
`Fork/Join`: `jdk7`中提供的一个并行计算框架，其原理和`MapReduce`相同

> `jdk7`发布时间晚于`MapReduce`，可能是借鉴了`MapReduce`的思想

#### 3. 一个例子

这里使用一个累加的任务作为演示

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ForkJoinTest extends RecursiveTask<Long> {
    private long start;
    private long end;

    private long limit;

    public ForkJoinTest(long start, long end, long limit) {
        this.start = start;
        this.end = end;
        this.limit = limit;
    }

    @Override
    protected Long compute() {
        if (end - start <= limit) {
            long result = 0;
            for (long i = start; i <= end; i++) {
                result += i;
            }
            return result;
        }
        long middle = (start + end) / 2;
        ForkJoinTest sub1 = new ForkJoinTest(start, middle, limit);
        ForkJoinTest sub2 = new ForkJoinTest(middle + 1, end, limit);
        invokeAll(sub1, sub2);
        return sub1.join() + sub2.join();
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        ForkJoinTest forkJoinTest = new ForkJoinTest(1, 1000000, 100000);
        Long r = pool.invoke(forkJoinTest);
        System.out.println(r);
    }
}
```

> `fork/join`框架提供`RecursiveAction`(无返回值)和`RecursiveTask`(有返回值)两种`Task`

#### 4. 参考

[Fork/Join](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)