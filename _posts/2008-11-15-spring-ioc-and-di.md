---
layout: post
title: 我对Spring IOC和DI的理解
categories: [编程, java, spring]
tags: [java, spring, ioc, di]
---

### IOC
IOC: Inversion of Control，中文译作`控制反转`

先看两段代码：

传统编码
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

ioc方式编码
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

说明：
* `控制`通常发生在对象的调用之间，在`传统编码`中，A对象需要调用B对象，则由A对象new一个B对象(发生在编译时)，在此过程中B对象的生命周期由A对象`控制`

* `反转`则是控制的权力由A对象转移到了第三方，在`IOC`方式中，B对象的创建和销毁由上层管理，A对象只是在运行时被动的接收了B对象

### DI
DI: Dependency Injection,中文译作`依赖注入`

还是以上面代码为例
* A对象依赖B对象，在`传统编码`中，A对象在编译期就依赖了B对象

* 在`IOC`方式中，A对象在运行期，由第三方`注入`B对象

### 总结
* `IOC`是一种思想，目的是把对象的依赖关系由编译时转移到运行时，这样的好处是消除了硬编码

* `DI`是`IOC`思想的一种实现，通过第三方控制对象依赖的注入，实现控制权的转移

* `面向接口编程`是DI的核心，即面向对象编程的基本特征：`抽象`