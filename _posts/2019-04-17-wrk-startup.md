---
layout: post
title: Wrk 试用
categories: [编程, linux, 性能测试]
tags: [wrk]
---


> 使用`ab`做压力测试，发现`tps`总是上不去，于是换`wrk`试试

#### 1. 下载安装

下载源码[wrk](https://github.com/wg/wrk)并编译
```
$ git clone https://github.com/wg/wrk.git
$ cd wrk
$ make
```

#### 2. 使用

```
$ ./wrk -t 16 -c 100 -d 10s http://192.168.10.100:8080/a.html 
Running 10s test @ http://10.142.90.23:7778/a.html
  16 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   738.17us  317.02us  12.71ms   93.18%
    Req/Sec     8.17k   735.53    13.57k    73.73%
  1308642 requests in 10.10s, 425.51MB read
Requests/sec: 129512.85
Transfer/sec:     42.11MB
```

* `-t`: 线程数，建议设为客户机`CPU`核数
* `-c`: 并发连接数，并发连接数/线程数=每个线程处理的连接数

> 8核主机`nginx`能压到`13W tps`

#### 参考

* [wrk](https://github.com/wg/wrk)