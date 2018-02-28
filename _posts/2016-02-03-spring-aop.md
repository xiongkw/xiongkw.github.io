---
layout: post
title: Spring aop的几种写法
categories: [编程, spring, java]
tags: [aop]
---
> `spring`中实现`aop`是非常简单的

#### 1. 普通代理模式
业务类 `MyService.java`
```java
public class MyService {

	public void doService() {
		System.out.println("do service");
	}
	
}
```

日志类 `MyLogger.java`
```java
public class MyLogger implements MethodBeforeAdvice {

	public void before(Method method, Object[] args, Object target) throws Throwable {
		System.out.println("my log");
	}
	
}
```

`aop-proxy.xml`
```xml
<bean id="service" class="com.my.spring.aop.MyService"></bean>

<bean id="myLogger" class="com.my.spring.aop.MyLogger"></bean>

<!-- 定义切点 匹配*sleep方法 -->
<bean id="logPointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut">
    <property name="pattern" value=".*Service"></property>
</bean>

<!-- 切面 增强+切点结合 -->
<bean id="logAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="advice" ref="myLogger" />
    <property name="pointcut" ref="logPointcut" />
</bean>

<!-- 定义代理对象 -->
<bean id="serviceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="service" />
    <property name="interceptorNames" value="logAdvisor" />
</bean>
```
> 普通代理模式的缺点：需要为每个业务bean都配置一个代理bean

`UnitTest`
```java
@ContextConfiguration(locations = { "classpath*:aop-proxy.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopProxyTest {
	@Resource(name = "serviceProxy")
	private MyService service;

	@Test
	public void test() {
		service.doService();
	}

}
```

#### 2. 自动代理模式
业务类 `MyService.java`，同上

日志类 `MyLogger.java`，同上

`aop-autoProxy.xml`
```xml
<bean id="service" class="com.my.spring.aop.MyService"></bean>

<bean id="myLogger" class="com.my.spring.aop.MyLogger"></bean>

<!-- 配置切点和通知 -->
<bean id="sleepAdvisor"
    class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
    <property name="advice" ref="myLogger"></property>
    <property name="pattern" value=".*Service" />
</bean>

<!-- 自动代理配置 -->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
```

> 自动代理模式相比普通代理模式的优点是不需要为每个业务`bean`都配置一个代理`bean`

`UnitTest`
```java
@ContextConfiguration(locations = { "classpath*:aop-autoProxy.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopPojoTest {
	@Autowired
	private MyService service;

	@Test
	public void test() {
		service.doService();
	}

}
```

#### 3. aop:config配置
业务类 `MyService.java`，同上

日志类 `AopConfigLogger.java`
```java
@Aspect
public class AopConfigLogger {

	public void before() {
		System.out.println("aop config log");
	}
	
}
```

`aop-config.xml`
```xml
<bean id="service" class="com.my.spring.aop.MyService"></bean>

<bean id="aopConfigLogger" class="com.my.spring.aop.AopConfigLogger"></bean>

<aop:config proxy-target-class="true">
    <aop:aspect ref="aopConfigLogger">
        <aop:before method="before" pointcut="execution(* *.doService(..))" />
    </aop:aspect>
</aop:config>
```

`UnitTest`
```java
@ContextConfiguration(locations = { "classpath*:aop-config.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopConfigTest {
	@Autowired
	private MyService service;

	@Test
	public void test() {
		service.doService();
	}

}
```

#### 4. aspectj模式
业务类 `MyService.java`，同上

日志类 `AspectjLogger.java`
```java
@Aspect
public class AspectjLogger {
	
	@Pointcut("execution(* *.doService(..))")
	private void pointcut(){
		
	}
	
	@Before("pointcut()")
	public void advisor() {
		System.out.println("before do service");
	}

}
```

`aop-aspectj.xml`
```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>

<bean id="myService" class="com.my.spring.aop.MyService"/>

<bean id="aspectjLogger" class="com.my.spring.aop.AspectjLogger"/>
```

`UnitTest`
```java
@ContextConfiguration(locations = { "classpath*:aop-aspectj.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopAspectjTest {
	@Autowired
	private MyService service;

	@Test
	public void test() {
		service.doService();
	}

}
```

#### 5. 总结

* `spring aop`不管使用哪种写法，其本质都是生成一个代理对象，在代理方法中加入增强方法
* `spring aop`默认使用`jdk代理`(需要定义接口)生成代理对象，如果业务对象没有实现接口，则使用`cglib`生成代理对象
* 因为代理的特性，使得其无法增强通过`this`调用的方法和`private`方法，参考[Spring load-time-weaver用法]({{ site.url}}/2016/02/10/spring-ltw-useage/)

#### 6. 附：代理的简单原理
```java
//原类
class Service{
    public void doService(){
        
    }
}

//代理类，这里用继承来演示
class ProxyedService extends Service{
    private Helper helper;
    
    public void doService(){
        //加入增强功能
        helper.doSomething();
        //调用原对象实际方法
        super.doService();
    }
    
}
```

