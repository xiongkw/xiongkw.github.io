---
layout: post
title: Java中轮询的工具类实现
categories: [编程, java]
tags: []
---

>  在多线程编程中，提交一个任务后，我们通常会去轮询任务执行结果

#### 1. 轮询工具类实现

```java
public class PollingUtils {

    /**
     *
     * @param supplier 结果供应者
     * @param predicate 断言
     * @param maxRetryTimes 最大重试次数
     * @param rate 间隔
     * @param <T>
     * @return
     * @throws TimeoutException
     */
    public static <T> T poll(Supplier<T> supplier, Predicate<T> predicate, int maxRetryTimes, long rate) throws TimeoutException {
        T t = supplier.get();
        int retryTimes = 0;
        while (!predicate.test(t)) {
            if (retryTimes++ > maxRetryTimes) {
                break;
            }
            try {
                Thread.sleep(rate);
            } catch (InterruptedException ignored) {
            }
            t = supplier.get();
        }
        if (t == null) {
            throw new TimeoutException();
        }
        return t;
    }

}
```

#### 1. 使用方法

```java
public static void main(String[] args) {
    Holder<String> holder = new Holder();
    // 提交一个异步任务
    new Thread(()->{
        Thread.sleep(50000);
        holder.set("completed");
    }).start();    
    
    PollingUtils.poll(() -> {
        // 查询结果
        return holder.get();
    }, Objects::nonNull, 10, 300);
}
```
