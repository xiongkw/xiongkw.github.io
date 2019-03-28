---
layout: post
title: Mariadb ERROR 1045 (28000)
categories: [编程, linux, mariadb]
tags: []
---


> `CentOS`上新安装的`Mariadb`，创建一个用户却怎么也登录不上

#### 1. 创建用户

```
> CREATE USER 'foo'@'%' IDENTIFIED BY 'foo';
> flush privileges;
> exit

$ mysql -ufool -pfoo
ERROR 1045 (28000): Access denied for user 'foo'@'localhost' (using password: YES)
```

#### 2. 原因

`ERROR 1045 (28000)`的原因是密码错误，那么为什么会密码错误呢？

原因：用户`'foo'@'localhost'`登录时会优先匹配到匿名用户`''@'localhost'`，当然和匿名用户的密码不匹配了

#### 3. 解决办法

删除匿名用户

```
> delete from mysql.user where user='';
> flush privileges;
```

#### 4. 参考

* [MySQL ERROR 1045 (28000): Access denied for user 'bill'@'localhost' (using password: YES)](https://stackoverflow.com/questions/10299148/mysql-error-1045-28000-access-denied-for-user-billlocalhost-using-passw?r=SearchResults)