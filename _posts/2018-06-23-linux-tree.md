---
layout: post
title: Linux下使用tree命令查看资源树结构
categories: [编程, linux]
tags: [tree]
---


> `Linux`中常用`ls`查看资源，但`ls`只能展示一级目录，如果需要同时展示多级目录，就需要用到`tree`命令了

#### 1. 下载安装

下载安装包[The Tree Command for Linux Homepage](http://mama.indstate.edu/users/ice/tree/)

解压
```
$ tar zxvf tree-1.7.0.tgz

$ cd tree-1.7.0

$ make install

```

#### 2. 使用

使用`tree`命令查看`tree-1.7.0`目录的资源树结构

```
tree tree-1.7.0

tree-1.7.0
├── CHANGES
├── color.c
├── color.o
├── doc
│   ├── tree.1
│   ├── tree.1.fr
│   └── xml.dtd
├── hash.c
├── hash.o
├── html.c
├── html.o
├── INSTALL
├── json.c
├── json.o
├── LICENSE
├── Makefile
├── README
├── strverscmp.c
├── TODO
├── tree
├── tree.c
├── tree.h
├── tree.o
├── unix.c
├── unix.o
├── xml.c
└── xml.o
```

> `tree`命令会递归展开所有层级目录，可以结合`less`命令滚动显示，例如 `tree . | less`
> 更多使用方法参考`tree --help`

#### 3. 参考

[The Tree Command for Linux Homepage](http://mama.indstate.edu/users/ice/tree/)