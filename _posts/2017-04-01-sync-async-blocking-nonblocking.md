---
layout: post
title: 同步与异步&阻塞与非阻塞
categories: [编程, java]
tags: [多线程, 并发, io]
---

> 编码中经常碰到`同步`与`异步`、`阻塞`与`非阻塞`的概念，这两组概念通常说的是调用者与被调用者之间的事，大到程序调用操作系统内核，小到两个对象之前的调用      
> 本文通过代码演示来理清这几个概念

#### 1. 区别

*`同步异步`关注的是`消息`的`通知机制`，`阻塞与非阻塞`则是`等待消息`时的`状态`*

#### 2. 代码演示

中午懒得出去吃饭，就在网上订了一个鸡蛋炒饭（居然还不包邮），送餐小哥电话里说马上送到，于是我就坐下继续写代码...

`送餐小哥`
```java
public class LittleBrother{
    
    //订餐，不留电话也没关系，帮我放前台漂亮MM那里
    public String order(String order){
        //打包
        //送餐
        return "鸡蛋炒饭";
    }
    
    //这次留电话了，万一前台MM不认识我，拒收了咋办？
    public String order(String order, final String phone){
        new Thread(){
            public void run(){
                //打包
                //送餐
                //送到了，打电话
                call(phone);
            }
        }.start();
        return null;
     }
    
     //小哥你到哪里了？还要多久送到？
    public String getStatus(){
        return "在楼下等电梯，马上就到";
    }

}
```

`我`
```java
public class LittleCoder{
    
    public void coding(){
        //写啊写啊写代码，写出一行好代码
        //...
    }
    
    public void eat(String lunch){
        //鸡蛋炒饭是世界上最好的编程语言
        //滚蛋，java才是
        //赶紧吃完写代码
    }

}
```

#### 3. 同步阻塞
```java
public static void main(String[] args){
    brother.order("鸡蛋炒饭");
    //1.同步：主动等待午餐送到，当前线程主动等待方法调用结束即为同步
    //2.阻塞：等待午餐，没力气写代码，我被阻塞了
    
    //午餐送到，开吃
    I.eat();
}
```

#### 4. 同步非阻塞
```java
public static void main(String[] args){
    //订餐完了，不等，继续写代码
    new Thread(){
        public void run(){
            brother.order("鸡蛋炒饭");
        }
    }.start();
    
    //2.非阻塞：等待午餐的同时还要继续写代码，即没有被阻塞
    //写啊写啊写代码，为什么总是有写不完的代码?
    I.coding();
    
    //1.同步：我主动询问小哥送到没有，当前线程主动等待方法调用结束即为同步
    String message = brother.getStatus();
    //什么？ 半个小时还没等到电梯？
        
    //午餐终于送到了，饿晕...
    I.eat();
}
```

#### 5. 异步阻塞
```java
public static void main(String[] args){
    brother.order("鸡蛋炒饭", 14710241024);
    //1.异步：到了打我电话，当前线程被动等待方法调用结束即为异步
    //2.阻塞：啥也不干，就等电话，饿的没力气写代码，即被阻塞了
    //这里和同步的区别是：同步是主动打电话问小哥到了没有，异步是被动等待小哥打电话
    while(noCall){
        Thread.sleep(1000);
    }
    
    //午餐终于来了，开吃
    I.eat();
}
```

#### 6. 异步非阻塞
```java
public static void main(String[] args){
    brother.order("鸡蛋炒饭", 14710241024);
    //1.异步：到了打我电话，当前线程被动等待方法调用结束即为异步
    //2.非阻塞：等待午餐的过程中继续写代码，写代码这么重要的事不能被阻塞
    I.coding();
    //正在愉快的coding中，突然被小哥的call打断，愤怒中...
    //停止coding，虽然愤怒，但人是铁饭是钢，吃完再找你算账
    I.eat();
}
```

#### 7. 附

关于`java IO`中的几种模式
* `同步阻塞` [java中的bio socket编程]({{ site.url}}/2015/04/03/java-bio-socket/)
* `同步非阻塞` [java中的nio socket编程]({{ site.url}}/2015/04/04/java-nio-socket/)
* `IO多路复用` [java中的nio多路复用 socket编程]({{ site.url}}/2015/04/05/java-nio-socket-select/)
* `异步非阻塞` [java中的aio socket编程]({{ site.url}}/2015/04/06/java-aio-socket/)