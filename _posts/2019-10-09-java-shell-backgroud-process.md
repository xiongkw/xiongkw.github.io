---
layout: post
title: java中启动linux后台进程
categories: [编程, linux, java]
tags: [shell]
---

> `Java`中执行耗时的`shell`脚本，因为`jvm`自身不保证高可用，所以不能以异步方式执行，于是想到通过后台进程执行

#### 1. 运行后台进程 

````java
String cmd = "nohup bash -c '/app/bin/test.sh' &";
Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd});
````

说明：

* 使用`nohup`运行后台进程
* `java`中使用`bash -c`执行`shell`命令

#### 2. 记录后台进程PID以及exit code

```java
String cmd = "nohup bash -c '/app/bin/test.sh >/dev/null 2>&1 & PID=$!; " +
            "echo $PID > /app/bin/test.PID; " +
            "wait $PID; " +
            "echo $? > /app/bin/test.RC' &";
Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd});
```

说明：

* `L1`: 后台运行脚本并记录其进程`ID($!)`
* `L2`: 把`PID`写入文件
* `L3`: 使用`wait`等待进程运行完成
* `L4`: 获取进程退出码(`$?`)并写入文件
* `& PID=$!`: `&`有和`;`同样的功能，即分隔语句

#### 3. 参考

* [How to execute shell command from Java](https://www.mkyong.com/java/how-to-execute-shell-command-from-java/)
* [How can I get both the process id and the exit code from a bash script](https://stackoverflow.com/questions/9261397/how-can-i-get-both-the-process-id-and-the-exit-code-from-a-bash-script)