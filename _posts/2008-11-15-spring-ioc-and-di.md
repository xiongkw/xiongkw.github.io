---
layout: post
title: 我对Spring IOC和DI的理解
categories: [编程, java, spring]
tags: [ioc, di]
---

> `IOC: Inversion of Control`，中文译作`控制反转`

#### 1. 先看两段代码

##### 1.1 传统编码
```java
class A{
    //A依赖B
    private B b = new B();
    
    public void doSomething(){
        //a do something
        //...
        b.doSomething();
    }
    
    public static void main(String[] args){
      A a = new A();
      a.doSomething();
    }
}

```

> `private B b = new B();`: `A`类中硬编码依赖`B`

##### 1.2 ioc方式编码
```java
class A{
    //A依赖InterfaceB
    private InterfaceB b;
    
    public void setB(InterfaceB b){
        this.b = b;
    }
    
    public void doSomething(){
        //a do something
        //...
        b.doSomething();
    }
    
    public static void main(String[] args){
      A a = new A();
      InterfaceB b = new B();
      a.setB(b);
      a.doSomething();
    }
}
```

> 在编译期`A`只依赖`B`的接口，在运行期才会传入具体的`B`实现

#### 2. 说明

* `控制`通常发生在对象的调用之间，在`传统编码`中，`A`对象需要调用`B`对象，则由`A`对象`new`一个B对象(发生在编译时)，在此过程中`B`对象的生命周期由`A`对象`控制`

* `反转`则是控制的权力由`A`对象转移到了第三方，在`IOC`方式中，`B`对象的创建和销毁由上层管理，`A`对象只是在运行时被动的接收了`B`对象

#### 3. DI

`DI: Dependency Injection`,中文译作`依赖注入`

还是以上面代码为例：
* `A`对象依赖`B`对象，在`传统编码`中，`A`对象在编译期就依赖了`B`对象
* 在`IOC`方式中，`A`对象在运行期，由第三方`注入` `B`对象

#### 总结

* `IOC`是一种思想，目的是把对象的依赖关系由编译时转移到运行时，这样的好处是消除了`硬编码`
* `DI`是`IOC`思想的一种实现，通过第三方控制对象依赖的注入，实现控制权的转移

* `面向接口编程`是`DI`的核心，即面向对象编程的基本特征：`抽象`