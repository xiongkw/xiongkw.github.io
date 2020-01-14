---
layout: post
title: mysql日志和回滚
categories: [编程, linux, mysql]
tags: [回滚, 日志]
---

> 2020年刚开始就出了个小事故，某`mysql`数据库被执行了误操作

#### 1. 开启binlog

编辑my.cnf，重启mysql服务即可
```
[mysqld]
server-id=1
log-bin=mysql-bin
```

#### 2. 执行如下sql

```
> create database mytest;
> use `mytest`; 
> create table user( 
> id int auto_increment,
> name varchar(256),
> primary key (id) 
> );
> insert into user('name') values ('zhangsan'); 
```

#### 3. 查看binlog文件

通过`mysql`客户端查

```
> SHOW MASTER LOGS;
Log_name	File_size
mysql-bin.000001	177
mysql-bin.000002	3058745
mysql-bin.000003	813
```

#### 4. 查看binlog详情

通过`mysql`客户端查看(也可通过`mysqlbinlog`命令查看)

```
> SHOW BINLOG EVENTS IN 'mysql-bin.000003' FROM 0 LIMIT 0,100;

Log_name	Pos	Event_type	Server_id	End_log_pos	Info
mysql-bin.000003	4	Format_desc	1	123	Server ver: 5.7.26-log, Binlog ver: 4
mysql-bin.000003	123	Previous_gtids	1	154	 
mysql-bin.000003	154	Anonymous_Gtid	1	219	SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql-bin.000003	219	Query	1	319	create database mytest
mysql-bin.000003	319	Anonymous_Gtid	1	384	SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql-bin.000003	384	Query	1	541	use `mytest`; create table user( id int auto_increment, name varchar(256), primary key (id) )
mysql-bin.000003	541	Anonymous_Gtid	1	606	SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql-bin.000003	606	Query	1	680	BEGIN
mysql-bin.000003	680	Table_map	1	732	table_id: 108 (mytest.user)
mysql-bin.000003	732	Write_rows	1	782	table_id: 108 flags: STMT_END_F
mysql-bin.000003	782	Xid	1	813	COMMIT /* xid=12 */
```

> 能查到`create database mytest`和`create table`，还有`Write_rows(insert)`

#### 5. 执行update和delete

假设有人误操作，执行了`update`和`delete`
```
> update user set name='lisi';
> delete from user;
```
 
再次查看`binlog`，发现多了`Update_rows`和`Delete_rows`

```
mysql-bin.000003	813	Anonymous_Gtid	1	878	SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql-bin.000003	878	Query	1	952	BEGIN
mysql-bin.000003	952	Table_map	1	1004	table_id: 108 (mytest.user)
mysql-bin.000003	1004	Update_rows	1	1066	table_id: 108 flags: STMT_END_F
mysql-bin.000003	1066	Xid	1	1097	COMMIT /* xid=681 */
mysql-bin.000003	1097	Anonymous_Gtid	1	1162	SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql-bin.000003	1162	Query	1	1236	BEGIN
mysql-bin.000003	1236	Table_map	1	1288	table_id: 108 (mytest.user)
mysql-bin.000003	1288	Delete_rows	1	1334	table_id: 108 flags: STMT_END_F
mysql-bin.000003	1334	Xid	1	1365	COMMIT /* xid=692 */
```

#### 6. 恢复

##### 6.1 查出position

先查出需要恢复的`binlog position`，这里直接使用`insert`日志的起始`606-813`

##### 6.2 通过`mysqlbinlog`命令恢复

```
$ mysqlbinlog --start-position=606 --stop-position=813 --database=mytest mysql-bin.000003 | mysql -uroot -proot
```

##### 6.3 查看恢复结果

```
> select * from user;

id	name
1	zhangsan
```

同理，恢复`update`操作

```
$ mysqlbinlog --start-position=878 --stop-position=1097 --database=mytest mysql-bin.000003 | mysql -uroot -proot
```

查看结果

```
> select * from user;

id	name
1	lisi
```

##### 6.4 关于flush logs

再次查看binlog

```
> SHOW BINLOG EVENTS IN 'mysql-bin.000003' FROM 0 LIMIT 0,100;
```

发现恢复的两次操作也记录到了`binlog`中，所以一般在恢复之前刷新下日志，生成一个新文件，这样就不会更改已有的`binlog`文件

```
> flush logs;
```

#### 7. 总结

可以通过`binlog`恢复数据，但是需要满足两个条件：

* `数据快照`：`binlog`记录的是每一次具体操作，其恢复原理是基于某份数据快照执行这些操作
* `binlog`：一般会设置自动清理策略，如果被清理掉就无法回天了，所以一般`binlog`会配合定时数据备份使用

> 所以数据或`binlog`断片的情况是无法恢复的，例如误操作执行了`DELETE`，而`binlog`又没有记录原始的`INSERT`