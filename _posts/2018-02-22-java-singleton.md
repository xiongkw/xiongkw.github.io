---
layout: post
title: 一个singleton引发的血案
categories: [编程, java]
tags: [singleton, 多线程, 内存屏障]
---

> 十年前初入码行时学到的第一个设计模式就是`singleton`，当时以为这就是最简单的设计模式，现在才明白那时太天真了

#### 1. 关于可见性、有序性和原子性

> 参考[java中的可见性、有序性和原子性]({{ site.url}}/2016/06/28/java-visibility-ordering-atomic/)

#### 2. 关于singleton
> 单例模式是为确保一个类只有一个实例，并提供一个全局访问点的一种设计模式

其特点：
* 只有一个实例
* 提供访问其唯一实例的`public`方法  

#### 3. 延迟加载的singleton

以下是一个常见的单例写法：
```java
public class LazySingleton {
    private static LazySingleton instance;

    private LazySingleton() {
    }

    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();
        }
        return instance;
    }
}
```

> 该方式在单线程下正常，但多线程环境就可能会创建多个实例，原因在于可能有多个线程同时进入`if (instance == null) {`代码块

#### 4. 线程安全的singleton
```java
public class ThreadSafeSingleton {
    private static ThreadSafeSingleton instance;

    private ThreadSafeSingleton(){}

    public static ThreadSafeSingleton getInstance(){
        synchronized (ThreadSafeSingleton.class) {
            if (instance == null) {
                instance = new ThreadSafeSingleton();
            }
            return instance;
        }
    }
}
```

> 使用`synchronized`关键字同步`if`块，这样就能保证每次只有一个线程进入`if`块，但每次调用`getInstance`时的锁请求都会引起`线程上下文`切换，对系统资源消耗比较大

#### 5. 双检查的线程安全singleton
```java
public class DCLThreadSafeSingleton {
    private static DCLThreadSafeSingleton instance;

    private DCLThreadSafeSingleton() {
    }

    public static DCLThreadSafeSingleton getInstance() {
        if (instance != null) {
            return instance;
        }
        synchronized (DCLThreadSafeSingleton.class) {
            if (instance == null) {
                instance = new DCLThreadSafeSingleton();
            }
            return instance;
        }
    }
}
```

> 通过双检查(`DCL`)对`ThreadSafeSingleton`改进，这样当`instance`第一次初始化后，所有线程都不会进入`synchronized`代码块了   

但这里又引出一个新的问题，由于`instance = new DCLThreadSafeSingleton()`不是一个原子操作，其操作可分解为：
```
memory = allocate();   //1：分配对象的内存空间
ctorInstance(memory);  //2：初始化对象
instance = memory;     //3：设置instance指向刚分配的内存地址
```

> 由于编译器和`cpu`的`重排序`机制，无法保证`2`一定发生在`3`之前，所以有可能在线程`A`以`132`顺序运行到`3`时，线程`B`已经通过`if(instance != null)`获取了`instance`，而此时`2`尚未执行

#### 6. volatile的singleton
```java
public class VolatileThreadSafeSingleton {
    private static volatile VolatileThreadSafeSingleton instance;

    private VolatileThreadSafeSingleton() {
    }

    public static VolatileThreadSafeSingleton getInstance() {
        if (instance != null) {
            return instance;
        }
        synchronized (VolatileThreadSafeSingleton.class) {
            if (instance == null) {
                instance = new VolatileThreadSafeSingleton();
            }
            return instance;
        }
    }
}
```

> `volatile`能够的作用是保障`可见性`和`有序性`，即使得`5. 双检查`中的三个操作按顺序执行

#### 7. static的singleton
```java
public class StaticSingleton {
    private static StaticSingleton instance = new StaticSingleton();

    private StaticSingleton() {
    }

    public static StaticSingleton getInstance() {
        return instance;
    }
}
```

> `static`类型的属性会在类加载时初始化，因为在同一个`ClassLoader`中一个类只会加载(`ClassLoader#loadClass()`是线程安全的)一次，这样便可保证`singleton`，缺点是不可`延迟加载`

#### 9. 内部类的singleton
```java
public class InnerClassSingleton {
    private static class Holder{
        private static final InnerClassSingleton instance = new InnerClassSingleton();
    }

    private InnerClassSingleton(){}

    public InnerClassSingleton getInstance(){
        return Holder.instance;
    }
}

```

> 延迟加载的`StaticSingleton`在内部类中通过`static`属性变量持有一个`singleton`，由于`static`属性在类装载时才初始化，这样就实现了`延迟加载`

#### 8. final的singleton
```java
public class FinalSingleton {
    private static final FinalSingleton instance = new FinalSingleton();

    private FinalSingleton() {
    }

    public static FinalSingleton getInstance() {
        return instance;
    }
}

```

#### 10. 枚举的singleton
```java
public enum  EnumSingleton {
    INSTANCE;

    private EnumSingleton(){}

}

```

#### 附：spring中singleton的实现

查看`AbstractBeanFactory#getBean`源码

```java
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
    return doGetBean(name, requiredType, null, false);
}
```

`AbstractBeanFactory#doGetBean`

```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
    
    // ...
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

    // ...
}
```

`DefaultSingletonBeanRegistry#getSingleton()`

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "'beanName' must not be null");
    synchronized (this.singletonObjects) {
        // ...
    }
}
```

> 可以看到`spring`中使用`synchorinized`同步块保证多线程下的`singleton`