---
layout: post
title: Eureka和Zookeeper集群在CAP上的比较
categories: [编程, java]
tags: [eureka, zookeeper]
---


#### 1. CAP

`CAP`: `Consistency，Availability，Partition Tolerance`，是指在分布式系统中一致性、可用性、分区容错性最多只能同时满足两个

##### 1.1 Consistency
`Consistency`:一致性，数据一致更新，所有数据变动都是同步的，可分为如下几类：

* 强一致性（`strong consistency`）。任何时刻，任何用户都能读取到最近一次成功更新的数据。
* 单调一致性（`monotonic consistency`）。任何时刻，任何用户一旦读到某个数据在某次更新后的值，那么就不会再读到比这个值更旧的值。也就是说，可　　获取的数据顺序必是单调递增的。
* 会话一致性（`session consistency`）。任何用户在某次会话中，一旦读到某个数据在某次更新后的值，那么在本次会话中就不会再读到比这值更旧的值　　 会话一致性是在单调一致性的基础上进一步放松约束，只保证单个用户单个会话内的单调性，在不同用户或同一用户不同会话间则没有保障。示例case：php的　　session概念。
* 最终一致性（`eventual consistency`）。用户只能读到某次更新后的值，但系统保证数据将最终达到完全一致的状态，只是所需时间不能保障。
* 弱一致性（`weak consistency`）。用户无法在确定时间内读到最新更新的值。

##### 1.2 Availability

`Availability`：可用性，系统具有好的响应性能

##### 1.3 Partition tolerance

`Partition tolerance`：分区容错性，分区故障不会导致系统不可用

> 在分布式系统中，不可能因为某个分区故障就导致整个系统不可用，所以一般都会保障分区容错性，那么剩下的就是C和A的抉择了

#### 2. zookeeper

`ZAB`: ZooKeeper Atomic Broadcast，通过一`Leader`多`Follower`的模式，提供读写服务。客户端随机连接到不同的节点，读操作直接由节点返回，而写操作统一由`Leader`发起，等到过半节点确认才认为写成功

* `zookeeper`写操作由`Leader`同步到各`Follower`，所以能保证一致性，但由于其只需过半节点确认成功即认为写成功，所以仍无法保证强一致性
* `zookeeper`读操作由节点直接执行，所以也保障了其可用性，但在选举过程中或脑裂时无法提供服务，所以也不能保证完全可用

#### 3. eureka

`Eureka`基于AP原则实现，保障可用性，而牺牲了一致性

* `Eureka`读写操作都由当前节点直接执行，再由内部线程同步到其它节点，不能保证每个节点的数据都是一致的
* `Eureka`没有`Leader`的概念，所有节点都可独立提供读写操作，所以能保证可用性

#### 4. 总结

在微服务架构中常用`Eureka`或者`zookeeper`来实现服务注册和发现，二者都能很好的胜任，因其各自的侧重点不同，所以使用场景也略有不同

* `zookeeper`侧重一致性，但在极端情况会不可用，所以更适合对服务注册实时性要求高的场合
* `Eureka`侧重可用性，但在极端情况下不保证一致性，所以更适合高可用的场合