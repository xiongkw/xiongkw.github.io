---
layout: post
title: 同步与异步/阻塞与非阻塞
categories: [编程, java]
tags: [多线程, 并发, IO, java]
---

> 编码中经常碰到同步与异步、阻塞与非阻塞的概念，这两组概念也经常容易混淆。   
> 本文通过代码演示来理清这几个概念

### 区别
*同步异步关注的是消息的通知机制，阻塞与非阻塞则是等待消息时的状态*

## 代码演示
以下是一个订餐的例子：

中午懒得出去吃饭，就在网上订了一个鸡蛋炒饭（居然还不包邮），送餐小哥电话里说马上送到，于是我就坐下继续...

送餐小哥
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

我(当前线程)
```java
public class LittleCoder{
    
    public void coding(){
        //写代码啊写代码，写出一行好代码
        //...
    }
    
    public void eat(String lunch){
        //鸡蛋炒饭是世界上最好的编程语言
        //滚蛋，php才是
        //赶紧吃完写代码
    }

}
```

#### 同步阻塞
```java
main(){
    brother.order("鸡蛋炒饭");
    //1.同步：主动等待午餐送到，当前线程主动等待方法调用结束即为同步
    //2.阻塞：等待午餐，没力气写代码了，我被阻塞了
    
    //午餐送到，开吃
    I.eat();
}
```

#### 同步非阻塞
```java
main(){
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
    //什么？ 你在楼下等了半小时电梯？
        
    //午餐终于送到了，饿晕...
    I.eat();
}
```

#### 异步阻塞
```java
main(){
    brother.order("鸡蛋炒饭", 14710241024);
    //1.异步：到了打我电话，当前线程被动等待方法调用结束即为异步
    //2.阻塞：啥也不干，就等电话，饿的没力气写代码了，即被阻塞了
    //这里和同步的区别是：同步是主动打电话问小哥到了没有，异步是被动等待小哥打电话
    while(noCall){
        Thread.sleep(1000);
    }
    
    //午餐终于来了，开吃
    I.eat();
}
```

#### 异步非阻塞
```java
main(){
    brother.order("鸡蛋炒饭", 14710241024);
    //1.异步：到了打我电话，当前线程被动等待方法调用结束即为异步
    //2.非阻塞：等待午餐的过程中继续写代码，写代码这么重要的事不能被阻塞
    I.coding();
    //正在愉快的coding中，突然被小哥的call打断，愤怒中...
    //停止coding，虽然愤怒，但人是铁饭是钢，吃完再找你算账
    I.eat();
}
```