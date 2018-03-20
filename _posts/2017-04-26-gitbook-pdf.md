---
layout: post
title: GitBook导出PDF
categories: [编程]
tags: [gitbook, pdf]
---


> 最近非常流行使用`GitBook`编写文档，因为其语法简洁，又可以直接生成静态`html`，所以很容易搭建在线文档

#### 1. 安装gitbook-cli

```
npm install gitbook-cli -g
```

#### 2. 查看pdf command

```
gitbook help

pdf [book] [output]         build a book into an ebook file
```

#### 3. 生成pdf

```
gitbook pdf
```

提示错误

```
EbookError: Error during ebook generation: 'ebook-convert'
```

> 需要安装`ebook-convert`，下载[Calibre](https://calibre-ebook.com/download)安装并加入`PATH`

再次运行命令，成功了

```
gitbook pdf
```