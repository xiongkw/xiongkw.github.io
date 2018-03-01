---
layout: post
title: java内存溢出分析之-堆溢出
categories: [编程, java]
tags: [heap, oom, jvisualvm]
---

#### 1. 堆内存划分

`JVM`堆内存的划分(`jdk8`)
```
年轻代（New）：年轻代用来存放JVM刚分配的Java对象
    Eden：Eden用来存放JVM刚分配的对象
    Survivor1：
    Survivro2：两个Survivor空间一样大，当Eden中的对象经过垃圾回收没有被回收掉时，会在两个Survivor之间来回Copy，当满足某个条件，比如Copy次数，就会被Copy到Tenured。显然，Survivor只是增加了对象在年轻代中的逗留时间，增加了被垃圾回收的可能性。
年老代（Tenured)：年轻代中经过垃圾回收没有回收掉的对象将被Copy到年老代
```

> 参考[java内存模型]({{site.url}}/2017/06/27/java-memory/)

#### 2. 堆内存中的垃圾回收
```
当年轻代内存满时，会引发一次普通GC，该GC仅回收年轻代。需要强调的时，年轻代满是指Eden代满，Survivor满不会引发GC
当年老代满时会引发Full GC，Full GC将会同时回收年轻代、年老代
当永久代满时也会引发Full GC，会导致Class、Method元信息的卸载
```

> 参考[java中的GC]({{site.url}}/2017/07/01/java-gc/)

#### 3. 堆内存大小设置
```
-Xms512M //初始值
-Xmx2014M//最大值
-Xmn1024M//年轻代
```

#### 4. 一个例子

本例通过不断`new`出强引用的对象让堆内存溢出
```java
public class HeapOutOfMemory {

	public HeapOutOfMemory(String name) {
		super();
		this.name = name;
	}

	private String name;

	public static void main(String[] args) throws InterruptedException {
		List<HeapOutOfMemory> list = new ArrayList<>();
		for (long i = 0;; i++) {
			list.add(new HeapOutOfMemory(new String("Inst: ") + i));
			if (i % 100000 == 0) {
			    //每10w次休眠3s，便于dump
				Thread.sleep(3000);
			}
		}
	}

}
```

> 启动vm参数 `-verbose:gc -Xms50M -Xmx50M -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:\dump\heapoutofmemory.hprof`

*使用`jvisualvm dump`两次，注意间隔3s以上*

![]({{site.url}}/public/images/2017-06-28-java-heap-out-of-memory-jvisualvm-1.png)
> 可以看到`HeapOutOfMemory`的实例数占到`32%`

![]({{site.url}}/public/images/2017-06-28-java-heap-out-of-memory-jvisualvm-2.png)
> 通过两次`dump`比较看到`HeapOutOfMemory`实例数增长最多

*查看`HeapOutOfMemory`的实例视图*

![]({{site.url}}/public/images/2017-06-28-java-heap-out-of-memory-jvisualvm-3.png)
> 可以看到`HeapOutOfMemory`的实例被`ArrayList`所引用，这里可以确定`ArrayList`中的`HeapOutOfMemory`实例没有被GC收回，因为其被`ArrayList`引用

*关于排名前二的`char[]`和`String`*   
这哥俩表示很冤屈：“其实我俩是被`HeapOutOfMemory`引用的，如图”

![]({{site.url}}/public/images/2017-06-28-java-heap-out-of-memory-jvisualvm-4.png)

*回头看看`console`里打印了什么*
```
[GC (Allocation Failure) [DefNew: 13696K->1664K(15360K), 0.0154519 secs] 13696K->5867K(49536K), 0.0154999 secs] [Times: user=0.00 sys=0.02, real=0.01 secs] 
[GC (Allocation Failure) [DefNew: 15360K->1664K(15360K), 0.0218158 secs] 19563K->12207K(49536K), 0.0218433 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[GC (Allocation Failure) [DefNew: 15360K->1664K(15360K), 0.0216375 secs] 25903K->18561K(49536K), 0.0216646 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC (Allocation Failure) [DefNew: 15360K->1664K(15360K), 0.0208552 secs] 32257K->24850K(49536K), 0.0208813 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC (Allocation Failure) [DefNew: 15360K->1664K(15360K), 0.0163532 secs] 38546K->31041K(49536K), 0.0163798 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC (Allocation Failure) [DefNew: 15360K->15360K(15360K), 0.0000121 secs][Tenured: 29377K->34176K(34176K), 0.0807648 secs] 44737K->37527K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0808212 secs] [Times: user=0.08 sys=0.00, real=0.08 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0878719 secs] 49536K->42771K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0879036 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0813069 secs] 49535K->45910K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0813405 secs] [Times: user=0.08 sys=0.00, real=0.08 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.1090144 secs] 49535K->47569K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.1090462 secs] [Times: user=0.11 sys=0.00, real=0.11 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0869439 secs] 49536K->48501K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0869855 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0883067 secs] 49535K->48991K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0883757 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0932435 secs] 49535K->49249K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0932738 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0897258 secs] 49536K->49385K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0897772 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0881933 secs] 49536K->49456K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0882218 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0854730 secs] 49535K->49494K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0855028 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0814072 secs] 49535K->49513K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0814366 secs] [Times: user=0.08 sys=0.00, real=0.08 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0852416 secs] 49535K->49524K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0852719 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0877851 secs] 49535K->49529K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0878112 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0849020 secs] 49535K->49532K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0849281 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0881121 secs] 49536K->49534K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0881443 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0906514 secs] 49535K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0906841 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0855196 secs] 49535K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0855537 secs] [Times: user=0.08 sys=0.01, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0897002 secs] 49536K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0897347 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0871320 secs] 49536K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0871571 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0896661 secs] 49535K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0896979 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0862470 secs] 49535K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0862768 secs] [Times: user=0.09 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0863813 secs] 49535K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0864196 secs] [Times: user=0.08 sys=0.00, real=0.09 secs] 
[Full GC (Allocation Failure) [Tenured: 34176K->34176K(34176K), 0.0842092 secs] 49535K->49535K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0842325 secs] [Times: user=0.09 sys=0.00, real=0.08 secs] 
java.lang.OutOfMemoryError: Java heap space
Dumping heap to d:\dump\d:\dump\heapoutofmemory.hprof ...
[Full GC (Allocation Failure) [Tenured: 34176K->761K(34176K), 0.0125575 secs] 49535K->761K(49536K), [Metaspace: 2140K->2140K(4480K)], 0.0125911 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 def new generation   total 15360K, used 295K [0x04800000, 0x058a0000, 0x058a0000)
  eden space 13696K,   2% used [0x04800000, 0x04849cc8, 0x05560000)
  from space 1664K,   0% used [0x05700000, 0x05700000, 0x058a0000)
  to   space 1664K,   0% used [0x05560000, 0x05560000, 0x05700000)
 tenured generation   total 34176K, used 761K [0x058a0000, 0x07a00000, 0x07a00000)
   the space 34176K,   2% used [0x058a0000, 0x0595e578, 0x0595e600, 0x07a00000)
 Metaspace       used 2160K, capacity 2280K, committed 2368K, reserved 4480K
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOfRange(Arrays.java:3664)
	at java.lang.String.<init>(String.java:201)
	at java.lang.StringBuilder.toString(StringBuilder.java:407)
	at com.mywork.mem.HeapOutOfMemory.main(HeapOutOfMemory.java:21)

Process finished with exit code 1
```

> 发生了`6`次普通`GC`和n次`Full GC`

#### 5. 参考

* [java内存模型]({{site.url}}/2017/06/27/java-memory/)
* [java中的GC]({{site.url}}/2017/07/01/java-gc/)
