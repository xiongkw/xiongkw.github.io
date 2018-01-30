---
layout: post
title: Linux文件切割与合并
categories: [编程, linux]
tags: [split]
---

#### 1.文本
* 按文件大小切割
```
split -C 20M nohup.out nohuplog
```
* 按文本行数切割
```
split -l 1000 nohup.out nohuplog
```
            
#### 2. 二进制
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
