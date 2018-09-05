---
layout: post
title: Spring-cloud应用在docker容器中启动慢的问题
categories: [编程, java, docker]
tags: [spring-cloud]
---


> `spring-cloud`应用，在本地启动只需`30s`，但是发布到`docker`容器中却要`120s`

#### 1. 查看docker容器cpu占用情况

```sh
$ docker ps

$ docker status ${container_id}
```

> 发现在启动过程中`cpu`占用率一直保持在`200%`左右，而当前容器资源限制`cpu`为`2核`

#### 2. 测试不同容器规格下的情况

|容器规格   |   起动耗时(s)  |
| -------- | -------------------|
| 2核4G | 108 |
| 4核8G | 50 |
| 8核8G | 32 |
| 16核8G | 29 |

> 发现确实和`CPU`有关，那为什么在本地开发`2核8G`不会出现这种情况呢?

#### 3. 分析jvm线程

进入容器：

```sh
$ docker exec -it ${container_id} bash
```

找出jvm进程id：

```sh
$ jps -v
```

查看jvm线程占用cpu时间：

```sh
$ ps -mp pid -o THREAD,tid,time

USER     %CPU PRI SCNT WCHAN  USER SYSTEM   TID     TIME
root      0.9  19    - futex_    -      -    59 00:00:06
root      1.1  19    - futex_    -      -    60 00:00:08
root      0.9  19    - futex_    -      -    61 00:00:07
root      0.8  19    - futex_    -      -    62 00:00:06
root      1.2  19    - futex_    -      -    63 00:00:09
root      1.4  19    - futex_    -      -    64 00:00:11
root      1.1  19    - futex_    -      -    65 00:00:08
root      1.2  19    - futex_    -      -    66 00:00:09
root      0.9  19    - futex_    -      -    67 00:00:07
root      1.5  19    - futex_    -      -    68 00:00:11
root      1.6  19    - futex_    -      -    69 00:00:11
root      0.8  19    - futex_    -      -    70 00:00:06
```

发现ID为`59-70`的线程占用`cpu`时间都比较多`(6-11s)`

查看线程stack：

```sh
$ printf "%x\n" 59
3b

$ jstack pid | grep -30 0x3b

"C2 CompilerThread10" #15 daemon prio=9 os_prio=0 tid=0x00007f0478120800 nid=0x45 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread9" #14 daemon prio=9 os_prio=0 tid=0x00007f047811e800 nid=0x44 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread8" #13 daemon prio=9 os_prio=0 tid=0x00007f047811c800 nid=0x43 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread7" #12 daemon prio=9 os_prio=0 tid=0x00007f047811a800 nid=0x42 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread6" #11 daemon prio=9 os_prio=0 tid=0x00007f0478118800 nid=0x41 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread5" #10 daemon prio=9 os_prio=0 tid=0x00007f0478116800 nid=0x40 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread4" #9 daemon prio=9 os_prio=0 tid=0x00007f0478114000 nid=0x3f waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread3" #8 daemon prio=9 os_prio=0 tid=0x00007f0478112000 nid=0x3e waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread2" #7 daemon prio=9 os_prio=0 tid=0x00007f0478110000 nid=0x3d waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #6 daemon prio=9 os_prio=0 tid=0x00007f047810e000 nid=0x3c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #5 daemon prio=9 os_prio=0 tid=0x00007f047810b000 nid=0x3b waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```

> 发现这些耗时最长的全是`C2 CompilerThread`，参考[What is code cache and compiler threads C1 and C2](http://marjavamitjava.com/code-cache-compiler-threads-c1-c2/)

#### 4. 关于C2线程

`jdk`文档中有一个`CICompiler`线程数的参数：`-XX:CICompilerCount=<threads>`

> Sets the number of compiler threads to use for compilation. By default, the number of threads is set to 2 for the server JVM, to 1 for the client JVM, and it scales to the number of cores if tiered compilation is used.

也就是说正常情况下`C2`线程数应该大于等于`1`，且小于等于`CPU核心`，但是我们这里在`2核`的容器中却跑出了十多个`C2`线程，看来`docker`容器限定的`2核`对jvm不起作用，和[《docker容器中jvm内存问题》]({{ site.url}}/2018/08/07/docker-jvm-memory/)相同，jvm又读取了宿主机的CPU核心数

#### 5. 设置CICompiler线程数

推测可能就是`C2`线程过多，导致线程之间相互等待引起，于是尝试减少`C2`线程数

在`2核4G`规格下，设置容器环境变量`-XX:CICompilerCount=2`，再次运行，启动时间降到了`41s`

使用`4核4G`规格运行，并设置`-XX:CICompilerCount=4`时，启动时间降到了`33s`

> `CICompilerCount`包含了`C1`和`C2`线程

#### 6. 结论

* `docker`容器的`CPU`和`内存`资源限制对`jvm`无效
* 多线程并不一定就更快，也有可能会更慢

#### 7. 参考

* [What is code cache and compiler threads C1 and C2](http://marjavamitjava.com/code-cache-compiler-threads-c1-c2/)

* [Compilation Optimization](https://docs.oracle.com/javacomponents/jrockit-hotspot/migration-guide/comp-opt.htm#JRHMG142)