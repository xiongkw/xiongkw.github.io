---
layout: post
title: Spring中的事件监听机制
categories: [编程, java, spring]
tags: [event]
---


> `spring`提供了`事件驱动`框架

#### 1. 定义一个事件
```java
public class MyEvent extends ApplicationEvent{

    private String code;

    private String message;

    public MyEvent(String code, String message, Object source) {
        super(source);
        this.code = code;
        this.message= message;
    }
    
}
```

#### 2. 定义事件监听器
```java
public class MyEventListener implements ApplicationListener<MyEvent> {

    @Override
    public void onApplicationEvent(MyEvent event) {
        System.out.println("code:"+event.getCode()+", message: "+event.getMessage());
    }

}
```

#### 3. 事件发布
```java
@Component
public class MyBean {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void onMyEvent(String code, String message){
        publisher.publishEvent(new MyEvent(code, message, this));
    }
    
}
```