---
layout: post
title: Jvm占用CPU过高问题排查
categories: [编程, java]
tags: [jvm]
---

> 线上环境突然变慢，查看进程，发现主机负载和进程cpu很高

#### 1. 查看进程CPU占用情况

```
$ top -Hp 45459

top - 11:22:30 up 242 days, 23:24,  4 users,  load average: 27.73, 30.41, 31.72
Threads: 465 total,  43 running, 422 sleeping,   0 stopped,   0 zombie
%Cpu(s): 37.8 us,  0.8 sy,  0.0 ni, 61.3 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 19662662+total,  5928868 free, 61176920 used, 12952083+buff/cache
KiB Swap:        0 total,        0 free,        0 used. 12995242+avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                                                                              
45470 fool      20   0   38.4g   4.0g  11940 R 53.2  2.1 107:11.01 java                                                                                                                                                                 
45499 fool      20   0   38.4g   4.0g  11940 R 53.2  2.1 107:11.39 java                                                                                                                                                                 
45462 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.21 java                                                                                                                                                                 
45463 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.34 java                                                                                                                                                                 
45464 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:09.57 java                                                                                                                                                                 
45465 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:08.71 java                                                                                                                                                                 
45466 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:12.31 java                                                                                                                                                                 
45471 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.60 java                                                                                                                                                                 
45472 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.41 java                                                                                                                                                                 
45474 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.30 java                                                                                                                                                                 
45475 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:09.91 java                                                                                                                                                                 
45476 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:12.32 java                                                                                                                                                                 
45477 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.17 java                                                                                                                                                                 
45478 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.51 java                                                                                                                                                                 
45479 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:08.24 java                                                                                                                                                                 
45481 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:09.75 java                                                                                                                                                                 
45482 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.14 java                                                                                                                                                                 
45483 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.84 java                                                                                                                                                                 
45485 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.29 java                                                                                                                                                                 
45486 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:12.20 java                                                                                                                                                                 
45487 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.72 java                                                                                                                                                                 
45488 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:13.25 java                                                                                                                                                                 
45491 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.27 java                                                                                                                                                                 
45492 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.60 java                                                                                                                                                                 
45494 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.93 java                                                                                                                                                                 
45495 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:09.73 java                                                                                                                                                                 
45496 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.74 java                                                                                                                                                                 
45498 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:12.68 java                                                                                                                                                                 
45500 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:10.18 java                                                                                                                                                                 
45502 fool      20   0   38.4g   4.0g  11940 R 52.8  2.1 107:11.21 java                                                                                                                                                                 
45461 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:12.04 java                                                                                                                                                                 
45467 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:10.42 java                                                                                                                                                                 
45468 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:11.81 java                                                                                                                                                                 
45469 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:11.00 java                                                                                                                                                                 
45480 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:11.88 java                                                                                                                                                                 
45484 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:12.14 java                                                                                                                                                                 
45489 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:10.68 java                                                                                                                                                                 
45490 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:12.56 java                                                                                                                                                                 
45493 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:11.00 java                                                                                                                                                                 
45497 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:12.22 java                                                                                                                                                                 
45501 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:10.35 java                                                                                                                                                                 
45503 fool      20   0   38.4g   4.0g  11940 R 52.5  2.1 107:11.83 java                                                                                                                                                                 
45473 fool      20   0   38.4g   4.0g  11940 R 52.2  2.1 107:08.64 java                                                                                                                                                                 
45505 fool      20   0   38.4g   4.0g  11940 S 22.9  2.1  97:40.29 java                                                                                                                                                                 
39053 fool      20   0   38.4g   4.0g  11940 S  5.3  2.1   1:56.21 java                                                                                                                                                                 
26779 fool      20   0   38.4g   4.0g  11940 S  5.0  2.1   0:25.03 java                                                                                                                                                                 
26783 fool      20   0   38.4g   4.0g  11940 S  4.7  2.1   0:25.52 java                                                                                                                                                                 
32769 fool      20   0   38.4g   4.0g  11940 S  4.3  2.1  56:49.13 java                                                                                                                                                                 
25684 fool      20   0   38.4g   4.0g  11940 S  4.3  2.1   0:26.48 java                                                                                                                                                                 
26152 fool      20   0   38.4g   4.0g  11940 S  4.3  2.1   0:26.12 java                                                                                                                                                                 
```

#### 2. 查看线程详情

```
$ jstack 45459 |grep $(printf "%x" 45472)
"pool-5-thread-22" #4086 prio=5 os_prio=0 tid=0x00007f05e8b1a000 nid=0x8642 waiting on condition [0x00007f06dd9dc000]
"GC task thread#11 (ParallelGC)" os_prio=0 tid=0x00007f0760033000 nid=0xb1a0 runnable 
```

> 发现占用CPU高的线程都是GC线程

#### 3. 查看GC详情

```
$ jstat -gcutil 45459 2000
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00   0.00 100.00 100.00  88.34  84.29 211557 6042.850  7272 6357.071 12399.921
  0.00   0.00 100.00 100.00  88.34  84.29 211557 6042.850  7272 6357.071 12399.921
  0.00   0.00  23.35 100.00  88.34  84.29 211557 6042.850  7274 6360.958 12403.807
  0.00   0.00  25.47 100.00  88.34  84.29 211557 6042.850  7276 6362.682 12405.532
  0.00   0.00 100.00 100.00  88.34  84.29 211557 6042.850  7278 6363.709 12406.559
```

> 发现老年代空间满了，导致不停的`FullGC`，可进一步分析堆内存

解释：

* S0: Survivor space 0 utilization as a percentage of the space's current capacity. 幸存者区0
* S1: Survivor space 1 utilization as a percentage of the space's current capacity. 幸存者区1
* E: Eden space utilization as a percentage of the space's current capacity. 伊甸园区
* O: Old space utilization as a percentage of the space's current capacity. 老年代
* M: Metaspace utilization as a percentage of the space's current capacity. 元空间
* CCS: Compressed class space utilization as a percentage. 压缩类空间利用率为百分比
* YGC: Number of young generation GC events. 年轻一代GC事件的数量
* YGCT: Young generation garbage collection time. 年轻一代垃圾收集时间
* FGC: Number of full GC events. 完整的GC事件的数量
* FGCT: Full garbage collection time. 完全垃圾收集时间
* GCT: Total garbage collection time. 垃圾回收总时间

#### 4. 查看堆内存使用情况

```
$ jmap -heap 45459
$ jmap -histo 45459
$ jmap -dump:format=b,file=dump.hprof 45459
```

#### 5. 分析dump文件

可使用`jdk`自带工具`jvisualvm`

#### 6. 参考
* [java内存溢出分析之-堆溢出]({{site.url}}/2017/06/28/java-heap-out-of-memory-jvisualvm/)
* [jstack](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)
* [jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)
* [jmap](https://www.jianshu.com/p/c52ffaca40a5)
