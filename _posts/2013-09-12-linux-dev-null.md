---
layout: post
title: linux中的重定向和/dev/null和2>&1
categories: [编程, linux]
tags: [linux]
---

> linux中经常碰到的 command >/dev/null 2>&1的用法

### 1. linux中的 /dev/null
> 在类Unix系统中，/dev/null，或称空设备，是一个特殊的设备文件，它丢弃一切写入其中的数据（但报告写入操作成功），读取它则会立即得到一个EOF。
  在程序员行话，尤其是Unix行话中，/dev/null 被称为位桶(bit bucket)或者黑洞(black hole)。空设备通常被用于丢弃不需要的输出流，或作为用于输入流的空文件。这些操作通常由重定向完成。
  
### 2.关于0、1、2
> 0、1和2分别表示标准输入、标准输出和标准错误信息输出

### 3. linux中的重定向
```
>               输出重定向到一个文件或设备 覆盖原来的文件
>!              输出重定向到一个文件或设备 强制覆盖原来的文件
>>              输出重定向到一个文件或设备 追加原来的文件
<               输入重定向到一个程序 
```

### 4. 如何理解 command >/dev/null 2>&1

> command会产生标准输出和标准错误两个输出，即1和2

* 先看`>/dev/null`，完整的写法应该是`1>/dev/null`，因为默认即是`1`所以通常省略。意思是把标准输出重定向到/dev/null   
* `2>&1` 把标准错误输出重定向到1(标准输出)

结合起来即把标准输出重定向到/dev/null，再把标准错误输出重定向到标准输出，即完全抛弃标准输出和标准错误输出

### 5. 关于 command 2>&1 1>/dev/null
`2>&1 1>/dev/null`不等于`1>/dev/null 2>&1`

```
错误信息仍然输出到了屏幕
[xx@xx ~]$ ifconfig -xx 2>&1 1>/dev/null
ifconfig: option `-xx' not recognised.
ifconfig: `--help' gives usage information.

错误信息没有输出到屏幕
[xx@xx ~]$ ifconfig -xx 1>/dev/null 2>&1

```

怎么理解？   
类似C语言中的指针，2指向1的地址，1指向null，这两个操作顺序颠倒后结果是不同的
```
int a=1;
int b=a;
a=2;
//这里b的值为1
b=a;
//这里b的值为2
```