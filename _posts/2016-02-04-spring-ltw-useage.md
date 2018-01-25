---
layout: post
title: Spring load-time-weaver 用法
categories: [编程, spring, java]
tags: [aop, ltw]
---

> spring load-time-weaver的简单用法

业务类 MyService.java
```java
public class MyService {

	public void doService() {
		System.out.println("do service");
	}
	
}
```

日志类 MyLogger.java
```java
@Aspect
public class LoggerAspect {

    @Before("execution(void *.privateService(..))")
    public void beforePrivateService(){
    	System.out.println("before private service...");
    }
}
```

META-INF/aop.xml
```xml
<aspectj>

	<weaver options="-verbose -showWeaveInfo -debug">
		<!-- include weaver target -->
		<include within="com.my.spring.aop.*" />
		<!-- Aspect也要include ?? -->
		<include within="com.my.spring.aop.weave.*" />
	</weaver>

	<aspects>
		<aspect name="com.my.spring.aop.weave.LoggerAspect" />
	</aspects>

</aspectj>
```

aop-weaver.xml
```xml
<context:load-time-weaver/>

<bean id="myService" class="com.my.spring.aop.MyService"></bean>
```

UnitTest
```java
/**
* 运行时需加入javaagent参数
* -javaagent:spring-instrument-x.x.x.jar
*/
@ContextConfiguration(locations = { "classpath*:aop-weaver.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopAspectjweaverTest {
	@Autowired
	private ApplicationContext context;

	@Test
	public void test() {
		MyService service = context.getBean(MyService.class);
		service.doPrivateService();
	}

}
```

*注意*

以下写法是不起作用的，原因请参考[Spring load-time-weaver原理]({{ site.url }}/2016/02/10/spring-ltw-source/)
```java
@ContextConfiguration(locations = { "classpath*:aop-weaver.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopAspectjweaverTest {
	@Autowired
	private MyService service;

	@Test
	public void test() {
		service.doPrivateService();
	}

}
```
