---
layout: post
title: Spring-boot热部署调试
categories: [编程, java, spring]
tags: [spring-boot, devtools, spring-loaded]
---


> `spring-boot`开发中，修改代码后需要重启才能生效

#### 1. 基于spring-boot-devtools的伪热部署

增加`spring-boot-devtools`依赖，运行即可
```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-devtools</artifactId>
      <scope>provided</scope>
</dependency>
```

> 从`console`日志可以发现每次修改代码并编译后，`spring-boot`重启了，这种通过重启程序的方式并不能算真正的热部署

#### 2. 基于spring-loaded

修改`spring-boot-maven-plugin`配置

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <dependencies>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>springloaded</artifactId>
      <version>1.2.8.RELEASE</version>
    </dependency>
    </dependencies>
</plugin>
```

启动方式有两种：

* 使用`(eclipse|idea)maven`插件运行: `mvn spring-boot:run`
* 直接运行`main`函数: `java -javaagent:D:/springloaded-1.2.8.RELEASE.jar -noverify`

#### 3. profile的结合使用

使用`main`函数运行指定`profile`，可使用参数`-Dspring.profiles.active`，例如：

```
-Dspring.profiles.active=dev -javaagent:D:/springloaded-1.2.8.RELEASE.jar -noverify
```

使用`maven`插件结合`spring-loaded`运行时，`-Dspring.profiles.active`不再起作用，但是不使用`spring-loaded`的情况下是正常的，查看`spring-boot-maven-plugin`源码：

```java
public abstract class AbstractRunMojo extends AbstractDependencyFilterMojo {

	/**
	 * Path to agent jar. NOTE: the use of agents means that processes will be started by
	 * forking a new JVM.
	 * @since 1.0
	 */
	@Parameter(property = "run.agent")
	private File[] agent;

	protected boolean isFork() {
		return (Boolean.TRUE.equals(this.fork)
				|| (this.fork == null && enableForkByDefault()));
	}

	private void run(String startClassName)
			throws MojoExecutionException, MojoFailureException {
		findAgent();
		boolean fork = isFork();
		this.project.getProperties().setProperty("_spring.boot.fork.enabled",
				Boolean.toString(fork));
		if (fork) {
			doRunWithForkedJvm(startClassName);
		}
		else {
			logDisabledFork();
			runWithMavenJvm(startClassName, resolveApplicationArguments().asArray());
		}
	}
}
```

> 原因是使用`agent`即`spring-loaded`后会`fork`出一个新的`jvm`，所以在`mvn`命令中通过`-D`指定的系统变量全部无效


解决办法：使用`-Dspring-boot.run.profiles`指定，参考[Specify active profiles](https://docs.spring.io/spring-boot/docs/1.5.9.RELEASE/maven-plugin/examples/run-profiles.html)

```
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

#### 4. 参考

* [Spring Boot Maven Plugin](https://docs.spring.io/spring-boot/docs/1.5.9.RELEASE/maven-plugin/)

* [Spring-Loaded](https://github.com/spring-projects/spring-loaded)