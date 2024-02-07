---
layout: post
title: Linux时间同步
categories: [编程, linux]
tags: []
---

>  

#### 1. ntpdate

能通公网的情况下最简单的方法是使用ntpdate命令

```
$ ntpdate time.nist.gov
```

推荐的时间同步服务器

```
time.nist.gov
time.nuri.net
0.asia.pool.ntp.org
1.asia.pool.ntp.org
2.asia.pool.ntp.org
3.asia.pool.ntp.org
```

#### 2. date命令

服务器使用http代理时，也可通过wget命令获取时间，但会有延迟

```
$ date -s "$(wget --no-cache -S -O /dev/null baidu.com 2>&1 | sed -n -e '/ *Date: */ {' -e s///p -e q -e '}')"
```
