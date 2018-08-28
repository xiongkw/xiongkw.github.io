---
layout: post
title: docker容器中jvm内存问题
categories: [编程, java, docker]
tags: [oom]
---


> 使用`docker`容器运行`spring-cloud`微服务时，偶尔发现容器会卡死

#### 1. 查看系统日志

`/var/log/messages`

```
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: [ pid ]   uid  tgid total_vm      rss nr_ptes swapents oom_score_adj name
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: [49297]     0 49297      255        1       4        0          -998 pause
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: [49420]     0 49420   104747    93279     206        0          -998 prometheus
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: Memory cgroup out of memory: Kill process 49297 (pause) score 0 or sacrifice child
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: Killed process 49297 (pause) total-vm:1020kB, anon-rss:4kB, file-rss:0kB
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: prometheus invoked oom-killer: gfp_mask=0xd0, order=0, oom_score_adj=-998
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: prometheus cpuset=6c32714aac70d0545f5601948c0cc954e5de945fa594de5afaf48bc01316d620 mems_allowed=0-3
Aug  7 15:52:13 A5-302-ZTE-R8500G3-080 kernel: CPU: 15 PID: 49420 Comm: prometheus Tainted: G    B      OE  ------------ T 3.10.0-327.el7.x86_64 #1
```

> 可以看到`docker`进程因为内存超出限制而被`kill`了

#### 2. 查看jvm堆内存

进入`docker`容器，查看`jvm heap`使用情况

```
[root@mytest-5d76d6dcd-756g9 /]# jmap -heap 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.112-b15

using thread-local object allocation.
Parallel GC with 43 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 32210157568 (30718.0MB)
   NewSize                  = 715653120 (682.5MB)
   MaxNewSize               = 10736369664 (10239.0MB)
   OldSize                  = 1431830528 (1365.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 3822583808 (3645.5MB)
   used     = 1908272728 (1819.8706893920898MB)
   free     = 1914311080 (1825.6293106079102MB)
   49.921017402059796% used
From Space:
   capacity = 25165824 (24.0MB)
   used     = 0 (0.0MB)
   free     = 25165824 (24.0MB)
   0.0% used
To Space:
   capacity = 234881024 (224.0MB)
   used     = 0 (0.0MB)
   free     = 234881024 (224.0MB)
   0.0% used
PS Old Generation
   capacity = 1028128768 (980.5MB)
   used     = 49215584 (46.935638427734375MB)
   free     = 978913184 (933.5643615722656MB)
   4.78690855968734% used
```

> `MaxHeapSize              = 32210157568 (30718.0MB)`可以看到`jvm`最大堆内存为`30G`，但是`docker`容器内存限制才`4G`，这里说明`jvm`的默认堆内存是以`宿主机`为准，而不是`docker容器`

#### 3. 设置容器jvm内存

设置`jvm`参数`"-Xms1g -Xmx2g -XX:PermSize=256m -XX:MaxPermSize=1g"`，再次查看`jvm堆内存`

```
[root@aep-template-mybatis001-6974bb464c-bpsv6 /]# jmap -heap 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.112-b15

using thread-local object allocation.
Parallel GC with 43 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 357564416 (341.0MB)
   MaxNewSize               = 715653120 (682.5MB)
   OldSize                  = 716177408 (683.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 402653184 (384.0MB)
   used     = 47805800 (45.591163635253906MB)
   free     = 354847384 (338.4088363647461MB)
   11.872698863347372% used
From Space:
   capacity = 145227776 (138.5MB)
   used     = 0 (0.0MB)
   free     = 145227776 (138.5MB)
   0.0% used
To Space:
   capacity = 138412032 (132.0MB)
   used     = 0 (0.0MB)
   free     = 138412032 (132.0MB)
   0.0% used
PS Old Generation
   capacity = 716177408 (683.0MB)
   used     = 51899144 (49.49488067626953MB)
   free     = 664278264 (633.5051193237305MB)
   7.2466882395709415% used
```

> 这里`Eden Space: used`比`2`中要少很多，原因是发生了`GC`，而`2`中因为没有超出`capacity`，所以不会引起`GC`

#### 4. jdk8支持

设置jvm内存设置可以解决上面的问题，但是总感觉很别扭，幸好`jdk8`提供了更好的方法：

> `jdk8u131+`以后可以通过设置参数`-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`，让jvm能查看到`docker`容器的内存限制，当容器内存达到上限时，同样会引进`jvm GC`