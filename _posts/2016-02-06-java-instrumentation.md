---
layout: post
title: Java Instrumentation
categories: [编程, spring, ltw, java]
tags: [spring, ltw]
---

> `spring load-time-weaver 用法`一文中在运行测试类时需要加入参数`-javaagent`

*Java Instrumentation指的是可以用独立于应用程序之外的代理（agent）程序来监测和协助运行在JVM上的应用程序。这种监测和协助包括但不限于获取JVM运行时状态，替换和修改类定义等。Instrumentation 的最大作用就是类定义的动态改变和操作。*

### 1.javaagent

看spring-instrument-x.x.x.jar!\META-INF\MANIFEST.MF源码
```yaml
Premain-Class: org.springframework.instrument.InstrumentationSavingAgent
Agent-Class: org.springframework.instrument.InstrumentationSavingAgent
```
> Premain-Class: 指定一个带有`premain`方法的类，jvm在启动main函数`之前`会调用该方法，@since jdk1.5   
> Agent-Class: 指定一个带有`agentmain`方法的类，用于`Attach API`方式，@since jdk1.6

### 2. agent写法
看spring-instrument-x.x.x.jar!\org\springframework\instrument\InstrumentationSavingAgent.class
```java
public class InstrumentationSavingAgent {

	private static volatile Instrumentation instrumentation;


	/**
	 * Save the {@link Instrumentation} interface exposed by the JVM.
	 */
	public static void premain(String agentArgs, Instrumentation inst) {
		instrumentation = inst;
	}

	/**
	 * Save the {@link Instrumentation} interface exposed by the JVM.
	 * This method is required to dynamically load this Agent with the Attach API.
	 */
	public static void agentmain(String agentArgs, Instrumentation inst) {
		instrumentation = inst;
	}

	public static Instrumentation getInstrumentation() {
		return instrumentation;
	}

}
```

### 3.关于`Attach API`
* 在jdk1.5时代只能使用`premain`方式，此方式的缺点是必须在启动时加入javaagent参数，并且在main函数启动前就加载代理类
* `Attach API`提供动态(即程序在运行中)加载代理类的功能

代码示例：
```java
    VirtualMachine vm = VirtualMachine.attach(id);
    vm.loadAgent("spring-instrument.jar");
```

### 4.instrumentation
instrumentation能做什么？看源码
```java
//增加一个ClassFileTransformer
void    addTransformer(ClassFileTransformer transformer);
//删除一个ClassFileTransformer
void    removeTransformer(ClassFileTransformer transformer);
...
```
ClassFileTransformer
```java
byte[]  transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
        throws IllegalClassFormatException;
```

*总结：*
* 通过`javaagent`或`Attatch API` 获取instrumentation对象
* instrumentation维护着一个字节码转换器列表
* 通过字节码转换来改变类的行为
* 常用的场景就是aop了，比如spring-ltw