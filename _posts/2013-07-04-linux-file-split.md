---
layout: post
title: Linux文件切割与合并
categories: [编程, linux]
tags: [split]
---

> `linux`中文件切割与合并

#### 1.文本文件
* 按文件大小切割：
```
split -C 20M nohup.out nohuplog
```

> `-C 20M`指定大小

* 按文本行数切割：
```
split -l 1000 nohup.out nohuplog
```

> `-l 1000`指定行数
            
#### 2. 二进制文件
```
split -b 20M jdk.tar.gz jdk
```

#### 3. 合并

文本：
```
cat nohuplog* > nohup.out
```

二进制：
```
cat jdk* > jdk.tar.gz
```
