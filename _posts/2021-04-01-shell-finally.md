---
layout: post
title: shell中的try-finally实现
categories: [编程, linux]
tags: [finally]
---

> shell中实现try-finally的效果

#### 1. 使用大括号

```
set -e
exit_code=0
{
    echo "try"
    ls aaa
    ls bbb
    echo "end"
} || {
    exit_code=$?
}

echo "finally"
exit $exit_code
```

> 以上写法的问题是即使`ls aaa`命令的`exit code`为`非0`，最终`exit code`仍然为`0`，原因`set -e`在这里无效

#### 2. 使用trap

优化的写法是使用`trap`命令，即在捕获到`EXIT`信号时执行`echo finally`命令

```
set -e
trap "echo finally" EXIT

echo "try"
ls aaa
ls bbb

```

常用的bash信号：

* HUP：编号1，脚本与所在的终端脱离联系。
* INT：编号2，用户按下 Ctrl + C，意图让脚本终止运行。
* QUIT：编号3，用户按下 Ctrl + 斜杠，意图退出脚本。
* KILL：编号9，该信号用于杀死进程。
* TERM：编号15，这是kill命令发出的默认信号。
* EXIT：编号0，这不是系统信号，而是 Bash 脚本特有的信号，不管什么情况，只要退出脚本就会产生。

#### 3. 参考

* [java中的nio socket编程]({{ site.url}}/2021/03/30/shell-finally/)
* [“Exit Trap” 让你的 Bash 脚本更稳固可靠](https://linux.cn/article-9639-1.html)