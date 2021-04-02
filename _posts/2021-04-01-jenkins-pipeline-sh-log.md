---
layout: post
title: jenkins-pipeline执行日志不全的问题
categories: [编程, linux]
tags: [jenkins]
---

> 使用jenkins-pipeline执行shell脚本，发现经常获取不到错误日志

#### 1. 现象

如下pipeline
```
node ('node1'){
    stage('BuildWithLabel') {
        try {
            sh '''
            set +x
            echo aaa
            ls file1
            ls file2
            echo haha
            '''
        } finally {
            cleanWs()
        }
    }
}
```

其中file1和file2都是不存在的，结果执行日志确没有错误信息

```
[Pipeline] Start of Pipeline
[Pipeline] node
Running on node1 in /home/jenkins/workspace/test
[Pipeline] {
[Pipeline] stage
[Pipeline] { (BuildWithLabel)
[Pipeline] sh
[Pipeline] cleanWs
[WS-CLEANUP] Deleting project workspace...[WS-CLEANUP] done
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
ERROR: script returned exit code 2
Finished: FAILURE
```


#### 2. 查看job日志

修改`sh`脚本，加一句`sleep 300`（为了能看到`durable-xxx`目录），再执行`pipeline`，进入`node1`节点`/home/jenkins/workspace/test@tmp`，查看`durable-xxx`目录

```
-rw-r--r-- 1 eadp eadp    9 4月   1 09:14 jenkins-log.txt
-rw-r--r-- 1 eadp eadp    1 4月   1 09:14 last-location.txt
-rw-r--r-- 1 eadp eadp  135 4月   1 09:14 script.sh
```

发现`script.sh`即是`pipeline`中定义的脚本，而`jenkins-log.txt`即是脚本执行的输出，最终看到`jenkins-log.txt`有完整的错误信息

#### 3. 原因分析

既然日志输出没问题，那可能就是`master`没有读取到完整的日志，再加上`durable-xxx`目录在`script.sh`执行完就被删除了，猜测可能`duralbe-xxx`目录被删除之时`master`还没有读取完`jenkins-log.txt`的内容

#### 4. 验证

使用`trap`命令达到`try-finally`的效果，即不管脚本执行成功与否，总是延时3秒再退出
```
node ('node1'){
    stage('BuildWithLabel') {
        try {
            sh '''
            set +x
            trap "sleep 3" EXIT
            echo aaa
            ls file1
            ls file2
            echo haha
            '''
        } finally {
            cleanWs()
        }
    }
}
```

这次能拿到完整日志了

```
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] node
Running on 10.190.190.36 in /home/eadp/jenkins/workspace/electrondesktop_CODE_BUILD-testxxx
[Pipeline] {
[Pipeline] stage
[Pipeline] { (BuildWithLabel)
[Pipeline] sh
+ set +x
aaa
ls: 无法访问'file1': 没有那个文件或目录
[Pipeline] cleanWs
[WS-CLEANUP] Deleting project workspace...[WS-CLEANUP] done
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
ERROR: script returned exit code 2
Finished: FAILURE
```

#### 5. 关于windows批处理

批处理即使出错也会继续执行后续所有命令，所以只需要在最后等待几秒再退出即可，批处理没有`sleep`，可以用`ping`命令模拟

```
ping -n 3 127.0.0.1>nul
```

#### 6. 参考

* [shell中的try-finally实现]({{ site.url}}/2021/03/30/shell-finally/)
* [“Exit Trap” 让你的 Bash 脚本更稳固可靠](https://linux.cn/article-9639-1.html)