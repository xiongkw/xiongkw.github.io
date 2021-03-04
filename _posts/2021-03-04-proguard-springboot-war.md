---
layout: post
title: 使用proguard混淆springboot-war包
categories: [编程, java]
tags: []
---

> 

#### 1. 工程结构

```
src/main/java
    |---com/webx
                |...
                |--Application.java
src/main/resources
    |---application.yml
pom.xml
```

#### 2. proguard配置
```
<build>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <showWarnings>true</showWarnings>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>com.github.wvengen</groupId>
                <artifactId>proguard-maven-plugin</artifactId>
                <version>2.3.1</version>
                <executions>
                    <execution>
                        <!-- 混淆时刻-->
                        <phase>process-classes</phase>
                        <goals>
                            <!-- 指定使用插件的混淆功能 -->
                            <goal>proguard</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <proguardVersion>6.2.2</proguardVersion>
                    <!-- 是否混淆-->
                    <obfuscate>true</obfuscate>
                    <!-- 指定生成文件分类 -->
                    <attachArtifactClassifier>pg</attachArtifactClassifier>
                    <libs>
                        <lib>${java.home}/lib/rt.jar</lib>
                        <lib>${java.home}/lib/jce.jar</lib>
                    </libs>
                    <!-- 需要混淆的字节码文件-->
                    <injar>classes</injar>
                    <outjar>pg-classes</outjar>
                    <!-- 输出目录-->
                    <outputDirectory>${project.build.directory}</outputDirectory>
                    <injarNotExistsSkip>true</injarNotExistsSkip>
                    <options>
                        <option>-dontshrink</option>
                        <option>-dontoptimize</option>
                        <!-- This option will replace all strings in reflections method invocations with new class names.
                             For example, invokes Class.forName('className')-->
                        <option>-adaptclassstrings</option>
                        <!-- This option will save all original annotations and etc. Otherwise all we be removed from files.-->
                        <option>-keepattributes
                            Exceptions,
                            InnerClasses,
                            Signature,
                            Deprecated,
                            SourceFile,
                            LineNumberTable,
                            *Annotation*,
                            EnclosingMethod
                        </option>
                        <!-- This option will save all original names in interfaces (without obfuscate).-->
                        <option>-keepnames interface **</option>
                        <!-- This option will save all original methods parameters in files defined in -keep sections,
                             otherwise all parameter names will be obfuscate.-->
                        <option>-keeppackagenames</option>
                        <option>-keepparameternames</option>
                        <!-- This option will save all original class files (without obfuscate) but obfuscate all
                             in domain and service packages.-->
                        <option>-keep @org.springframework.boot.autoconfigure.SpringBootApplication class * {*;}
                        </option>
                        <!-- This option ignore warnings such as duplicate class definitions and classes in incorrectly
                            named files-->
                        <option>-ignorewarnings</option>
                        <!-- This option will save all original class files (without obfuscate) in service package-->
                        <!--<option>-keep class com.slm.proguard.example.spring.boot.service { *; }</option>-->
                        <!-- This option will save all original interfaces files (without obfuscate) in all packages.-->
                        <option>-keep interface * extends * { *; }</option>
                        <option>-keepclassmembers enum * { *; }</option>
                        <option>-keepattributes Signature</option>
                        <!-- This option will save all original defined annotations in all class in all packages.-->
                        <option>-keepclassmembers class * {
                            @org.springframework.beans.factory.annotation.Autowired *;
                            @org.springframework.beans.factory.annotation.Value *;
                            }
                        </option>
                        <option>-keep class !com.webx.**,** { *; }</option>
                    </options>
                </configuration>
                <dependencies>
                    <!-- https://mvnrepository.com/artifact/net.sf.proguard/proguard-base -->
                    <dependency>
                        <groupId>net.sf.proguard</groupId>
                        <artifactId>proguard-base</artifactId>
                        <version>6.2.2</version>
                        <scope>runtime</scope>
                    </dependency>
                </dependencies>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>2.6</version>
                <configuration>
                    <warName>${project.artifactId}</warName>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                    <classesDirectory>${project.build.directory}/pg-classes</classesDirectory>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>1.5.14.RELEASE</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

> 思路：
> 1. 混淆classes文件夹内.class文件，输出到pg-classes目录
> 2. 打war包时指定classesDirectory为pg-classes
> 3. 通过spring-boot-maven-plugin插件打springbot war包

#### 3. 参考

* [ProGuard manual](https://www.guardsquare.com/en/products/proguard/manual/refcard)
* [proguard-spring-boot-example](https://github.com/seregaSLM/proguard-spring-boot-example/blob/master/pom.xml)
* [Obfuscating WAR file with Proguard](https://stackoverflow.com/questions/28690812/obfuscating-war-file-with-proguard)