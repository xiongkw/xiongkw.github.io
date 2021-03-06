---
layout: post
title: 使用maven-shade-plugin插件打包你的api依赖
categories: [编程, java]
tags: [maven, shade]
---


> 对于`api`设计者来说，尽可能减少第三方依赖是一个很重要的原则，因为你不知道项目会依赖到多少第三方`jar`包，而各种不同版本的jar及其传递依赖很可能成为`j2ee`开发的噩梦   
> 除此以外，一个简洁干净的`api`(不依赖任何第三方`jar`包)会让开发人员充满使用的欲望

#### 1. 关于第三方依赖

不可避免的第三方依赖   

> `java`的世界如此繁荣昌盛，正是因为有着如此庞大的第三方`api`，使得开发人员可以拿来即用，而不用重复的造轮子。此外如何正确的组合使用多种第三方`api`也是一个考验技术的标准。

#### 2. maven-shade-plugin
[](http://maven.apache.org/plugins/maven-shade-plugin/)

> This plugin provides the capability to package the artifact in an uber-jar, including its dependencies and to shade - i.e. rename - the packages of some of the dependencies.

`mave-shade-plugin`关注的是把`java`项目连同其依赖一起打成一个超级`jar`包，并且提供掩盖的功能

#### 3. 一个例子
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.0.0</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <minimizeJar>true</minimizeJar>
                        <relocations>
                            <!-- 把com.goole.gson包名转换为com.my.gson，即实现了第三方jar包的隐藏 -->
                            <relocation>
                                <pattern>com.google.gson</pattern>
                                <shadedPattern>com.my.gson</shadedPattern>
                            </relocation>
                        </relocations>
                        <artifactSet>
                            <excludes>
                                <!-- 排除不需要的依赖 -->
                                <exclude>org.slf4j:slf4j-api</exclude>
                            </excludes>
                        </artifactSet>
                        <filters>
                            <!-- 过滤不需要的资源文件 -->
                            <filter>
                                <artifact>com.google.code.gson:gson</artifact>
                                <excludes>
                                    <exclude>META-INF/**/*</exclude>
                                </excludes>
                            </filter>
                        </filters>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```