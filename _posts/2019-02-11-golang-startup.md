---
layout: post
title: Go语言入门
categories: [编程, go]
tags: [go]
---


> 新年开工第一天，折腾一下`go`语言

#### 1. 下载安装

[下载](https://golang.google.cn/dl/)

本文以`go1.11.5.windows-amd64.msi`为例，安装完成后会自动写入系统环境变量

检查是否安装成功
```
$ go version
go version go1.11.5 windows/amd64
```

#### 2. HelloWorld

编写`helloworld.go`
```go
package main

import "fmt"

func main() {
   fmt.Println("Hello, World!")
}
```

运行

```
$ go run helloworld.go
Hello, World!
```

#### 3. 基础语法

##### 3.1 源码结构

一个源文件由以下部分构成

```go
// 包名
package main

// 导入
import "fmt"

// 函数
func main() {
    // 语句
   fmt.Println("Hello, World!")
}
```

> `package main`的`func main`是固定的写法，表示一个应用的入口

##### 3.2 大括号`{`不可另起一行，如下会导致编译错误

```
func main()
{

}
```

##### 3.3 分号`;`不是必须的，类似`js`，默认一行就是一句，如果非要在一行写多句，可用分号隔开

```
fmt.Println("Hello, World!")
fmt.Println("Hello, World2!")
```

##### 3.4 使用关键字`var`声明静态类型的变量，可同时定义多个变量并赋值

```
var i int
var x,y int = 1,2 
```

##### 3.5 使用`:=`声明并初始化一个动态类型变量

```
i := 1
```

##### 3.6 使用`const`声明一个常量

```
const str string = "abc"
```

##### 3.7 返回多个值的函数

```
package main

import "fmt"

func swap(x, y string) (string, string) {
   return y, x
}

func main() {
   a, b := swap("Mahesh", "Kumar")
   fmt.Println(a, b)
}
```

#### 4. IDE

##### 4.1 下载安装LiteIDE

下载[LiteIDE](https://sourceforge.net/projects/liteide/files/)

解压即可

##### 4.2 环境配置

本文以`win64`为例

* 选择编译环境：

`工具`-`选择环境`-`win64`

* 修改`GOROOT`

`查看`-`选项`-`LiteEnv`，双击`win64.env`

```
GOROOT=d:\work\go
```

##### 4.3 创建工程

`文件`-`新建`-`Go1 Command Project`

生成的源文件`main.go`

```
// test project main.go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello World!")
}

```

##### 4.4 编译运行

* 编译 `CTRL+B`

* 运行 `CTRL+ALT+R`

* 编译并运行 `CTRL+R`

#### 5. 参考

* [Go 语言教程](http://www.runoob.com/go/go-tutorial.html)

* [Go 语言入门教程](http://c.biancheng.net/golang/)