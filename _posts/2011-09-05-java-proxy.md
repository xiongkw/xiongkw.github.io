---
layout: post
title: Java中的动态代理
categories: [编程, java]
tags: [proxy]
---


> `java`中的动态代理

#### 1. 场景 

员工请假必须找领导签字，但是领导一般都很忙，经常不在办公室，所以领导指定由其秘书代其签字

#### 2. Signer接口
```java
public interface Signer{
    void sign();
}
```

#### 2. 领导
```java
public class Leader implements Signer {

	public void sign() {
		// sign
	}

}
```

#### 3. InvocationHandler
```java
public class SecretaryHandler implements InvocationHandler {
	Signer signer;
	
	public LiuInvocationHandler(Signer signer) {
		this.signer = signer;
	}

	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		System.out.println("before");
		Object secretary = method.invoke(signer, args);
		System.out.println("after");
		return secretary;
	}

}

```

#### 4. main
```java
public static void main(String[] args){
    Signer leader = new Signer();
    Signer secretary = (Signer) Proxy.newProxyInstance(Main.class.getClassLoader(),
            new Class[] { Signer.class }, new SecretaryHandler(leader));
    secretary.sign();
}
```

#### 5. 总结

* `jdk动态代理`实际是生成了一个代理类并实现同样的接口，在方法实现中调用代理对象的方法
* `代理模式`和`装饰模式`实现上比较相似，其不同点是`代理`是原对象不方便处理而指定一个代理(比如我们办证时如果本从不方便去办证中心，可以通过公证指定某人代为办理)，而`装饰`则侧重于功能的增强