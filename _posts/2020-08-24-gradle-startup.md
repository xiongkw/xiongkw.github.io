---
layout: post
title: Gradle入门
categories: [编程, java]
tags: [gradle]
---

#### 1. 创建一个Gradle工程

本文直接使用`IntelliJ IDEA`创建一个`Gradle`工程，目录结构如下

```
.gradle //gradle安装文件会自动下载到这里
gradle  //wrapper文件夹
    |---wrapper
src
    |---main
        |---java
        |---resources
    |---test
        |---java
        |---resources
build.gradle    //构建脚本
gradlew     //gradlew shell脚本
gradlew.bat //gradlew bat脚本
settings.gradle //settings
```

#### 2. 构建

直接执行build命令即可

```
gradlew build
```

> 第一次执行时`gradlew`会自动下载`gradle`到`.gradle`文件夹

#### 3. 编写一个springboot例子

##### 3.1 修改build.gradle

```
plugins {
    id 'java'
    id 'war'
    id 'org.springframework.boot' version '1.5.14.RELEASE'
}

sourceCompatibility='1.8'

repositories {
    maven { url 'http://192.168.1.100:8081/nexus/content/groups/public' }
    mavenCentral()
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web:1.5.14.RELEASE'
    testCompile 'junit:junit:4.12'
}

```

> 添加maven私服，添加springboot和插件依赖

##### 3.2 编写Application.java

```
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

##### 3.3 编写一个controller

```
@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello(){
        return "Hello world";
    }
}
```

##### 3.3 编译

执行`gradlew build`，等待编译完成

```
build
    |---classes
    |---libs
        |---mytest-1.0.0-SNAPSHOT.war
        |---mytest-1.0.0-SNAPSHOT.war.original
```

#### 4. 参考

* [Gradle 教程](https://www.w3cschool.cn/gradle/6qo51htq.html)