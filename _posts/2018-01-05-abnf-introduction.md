---
layout: post
title: ABNF介绍
categories: [编程]
tags: [ABNF]
---

#### 1. 简介
* `BNF`: `Backus-Naur Form`(巴科斯范式), 是由 `John Backus` 和 `Peter Naur` 首先引入的用来描述计算机语言语法的符号集
* `ABNF`: `Augmented BNF`, 扩展的巴科斯范式

通俗讲，`ABNF`就是一种语法规范，类似正则表达式，用于定义某种具体的规则，例如描述一个至少6位长的密码

```
正则表达式：
.{0,6}

ABNF:
6*CHAR
```

#### 2. 语法

##### 2.1 规则定义

`=`定义一个规则

```
PASSWORD = 6*CHAR

mypassword = PASSWORD
```

> 注意规则名不区分大小写，如上`PASSWORD`和`password`对应同一个规则

##### 2.2 字符

`ABNF`中字符有两种定义方式

* `ASCII`

```
0x65
```

* 引号

```
"a"
```

> 需要注意的是引号定义的字符不区分大小写

##### 2.3 规则串联
 
` `(空格)用于串联规则

```
 num = 0x30 0x31; 匹配 01
```

##### 2.4 或选择

`/`用于`或`选择

```
num = 0x30 / 0x31; 匹配0|1
```
 
##### 2.5 范围取值

`-`用于范围取值

```
num = 0x30-0x32; 匹配0|1|2
```

##### 2.6 分组

`()`用于分组，同正则表达式

```
num = 0x30 (0x31 / 0x32) 0x33; 匹配013|023
```

##### 2.7 重复

`n*m`用于重复

```
rule = ...

num = 2rule; 重复2次
num = *2rule; 0-2次
num = 1*rule; 1-n次
num = 1*3rule; 1-3次

```

##### 2.8 注释
`;`用于注释

```
password = 6*CHAR; 定义一个至少6位字符的规则
```

#### 3. 核心规则

|名称   |   定义  |  说明 |
| ------- | -------------------- | ------------------------------ |
|ALPHA	  |%x41-5A / %x61-7A	 |大写和小写 ASCII 字母 (A-Z a-z) |
|DIGIT	  |%x30-39	|数字 (0-9)                                   |
|HEXDIG	| DIGIT / "A" / "B" / "C" / "D" / "E" / "F" |	十六进制数字 (0-9 A-F a-f)|
|DQUOTE	| %x22|	双引号|
|SP	| %x20 |	空格|
|HTAB	| %x09	| 水平tab|
|WSP	|SP / HTAB|	空格和水平tab|
|LWSP	|*(WSP / CRLF WSP)|	线性空白(晚于换行)           |
|VCHAR	|%x21-7E|	可见(打印)字符                      |
|CHAR	|%x01-7F|	任何 7-位 US-ASCII 字符，不包括 NUL|
|OCTET	|%x00-FF|	8 位数据     |
|CTL	|%x00-1F / %x7F|	控制字符|
|CR	|%x0D|	回车            |
|LF	|%x0A|	换行            |
|CRLF|	CR LF|	回车换行|
|BIT|	"0" / "1"| 比特0|1 |	 

#### 5. 参考资料

[Augmented BNF for Syntax Specifications: ABNF](https://tools.ietf.org/html/rfc5234)