---
layout: post
title: Spring-boot中的异步scheduler
categories: [编程, java, spring]
tags: [spring, scheduler]
---

> `spring-boot`中使用`@Scheduled`实现定时任务，发现同一个任务的`fixedRate`不起作用，即总是等到上一次调度完成后，才开始下一次调度

#### 1. 用法

如下例子：

```
@Component
public class MyTest {

    @Scheduled(fixedRateString = "1000")
    public void testFixRate() throws InterruptedException {
        long l = System.currentTimeMillis();
        System.out.println("FixRate begin, No: " + l);
        Thread.sleep(5000);
        System.out.println("FixRate end, No:" + l);
    }

    @Scheduled(cron = "* * * * * ?")
    public void testCron() throws InterruptedException {
        long l = System.currentTimeMillis();
        System.out.println("Cron begin, No: " + l);
        Thread.sleep(5000);
        System.out.println("Cron end, No:" + l);
    }

}
```

> 发现不仅两个任务是串行的，同一个任务的多次调度也是串行的

#### 2. 让两个任务并发执行

默认的`TaskExecutor`为单线程，所以改为多线程即可

```
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskExecutor());
    }

    @Bean(destroyMethod = "shutdown")
    public Executor taskExecutor() {
        return Executors.newScheduledThreadPool(8);
    }

}
```

#### 3. 让同一个任务的多次调度并发执行

改造成异步任务即可

```
@Scheduled(fixedRateString = "1000")
@Async
public void testFixRate() throws InterruptedException {

}

@Scheduled(cron = "* * * * * ?")
@Async
public void testCron() throws InterruptedException {

}
```

开启异步任务

```
@EnableAsync
public class MyApplication {

}
```