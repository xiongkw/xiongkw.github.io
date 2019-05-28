---
layout: post
title: maven批量修改父子模块的pom版本号
categories: [编程, java]
tags: [maven]
---


> 通常我们会用父子结构来组织一个大的`maven`工程，其父子模块`pom`版本号也会统一管理

#### 1. 修改为release版本

发版了，需要修改`1.0.0-SNAPSHOT`版本为`1.0.0`版本

```
$ mvn versions:set -DnewVersion=1.0.0
```

> 可以看到每个模块的`pom.xml`版本号已经变成了`1.0.0`，并且多出一个备份文件`pom.xml.versionsBackup`

#### 2. 提交

```
$ mvn versions:commit
```

> 提交完成后，备份文件`pom.xml.versionsBackup`消失了

#### 3. 关于备份文件

备份文件`pom.xml.versionsBackup`用于撤销更改，并且只能在提交之前

```
$ mvn versions:revert
```

#### 参考

* [Versions Maven Plugin](http://www.mojohaus.org/versions-maven-plugin/)