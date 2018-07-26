---
layout: post
title: Git 多用户commit时冲突的问题
categories: [编程, git]
tags: [commit]
---

> 项目组又双叒叕开了一个`gogs`，又双叒叕要申请账号，账号名一直用同一个`fool`，但是`email`给了一个不同的，然后开始愉快的写代码

#### 1. 写完代码了

```
git add *
git commit -m init

git push origin develop

Counting objects: 62, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (49/49), done.
Writing objects: 100% (62/62), 9.15 KiB | 1.52 MiB/s, done.
Total 62 (delta 22), reused 0 (delta 0)
remote: Authority [develop] passed.
remote:
remote: ERROR: Invalid user email on object e7d1391833a083e0b9605313a7f662257b18e79f:
remote:        Expecting "aaaaa@xx.com", got "bbbbb@xx.com"
remote:
remote: Please run the following commands and try again:
remote: > git config user.email "aaaaa@xx.com"
remote: > git commit --amend --author="fool <aaaaa@xx.com>"
remote:
remote: Gogs: Internal error

```

#### 2. 原因分析

通过错误提示可以看到原因是使用了不同的`email`提交，通过现象可以推测：

* `git clone`的时候只会校验账号而不会校验`email`，但是`push`的时候会校验`commit`中的`email`
* `git commit`的时候使用了`global`的`user和email`

#### 3. 解决方法

```
> git config user.email "aaaaa@xx.com"

> git commit --amend --author="fool <aaaaa@xx.com>"
```

##### 3.1 修改仓库用户
`git config user.email "aaaaa@xx.com"`
或者
```
git config -e

[user]
    email = aaaaa@xx.com
```

##### 3.2 修正提交者信息

```
git commit --amend --author="fool <aaaaa@xx.com>"
```

> 注意：`方法2`只能修改上一次`commit`，并且每次`commit`后都需要修正，这里推荐第一种方法，一劳永逸

#### 4. 附

使用指定账号`clone`的方法

```
http://username:password@192.168.1.100:9000/mytest.git
```