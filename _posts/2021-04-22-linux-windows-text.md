---
layout: post
title: Linux下的^M
categories: [编程, linux]
tags: []
---

> `windows`下的文本文件传到`linux`上会有`^M`字符的问题

#### 1. ^M产生的原因

> `windows`格式的文本文件，用回车换行`\r\n(0D 0A)`作为换行符，而`linux` 的则是以`\n(0A)` 作为换行符，所以`windows`格式的文件传到`linux`上换行符就会多出来一个`\r(0D)`显示为`^M`

#### 2. 在vi中显示^M

```
:e ++ff=unix %
```

#### 3. 使用vi删除^M

```
:%s/^M$//g
```
> 其中`^M`通过按键`CTRL+V`再`CTRL+M`输入

#### 4. 使用find和sed批量转换

```
$ find . -name '*.sh' -exec sed -i 's/^M$//g' {} \;
```