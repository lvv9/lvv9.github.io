# 一致性

## ACID

### Atomicity
原子性意味着无法完成最终提交时，数据库须丢弃或撤销那些局部完成的更改。
这里的原子性需要区分并发编程中的原子性，这个并没有描述多个线程试图访问相同的数据会发生什么情况，后者其实是由ACID的隔离性所定义。

### Consistency
一致性本质要求应用层来维护状态一致。应用程序可能借助数据库提供的原子性和隔离性，以达到一致性（即一致性是原子性与隔离性的充分不必要条件）。
字母C放在这里只是为了更顺口。

### Isolation
隔离级别见 [这里](https://liuweiqiang.me/2019/01/28/database-note.html)

### Durability
持久性，事务提交后就不会丢失（一般指硬盘等非易失性存储有记录）。

## CAP
> 正式定义的CAP定理范围很窄，它只考虑了一种一致性模型（即线性化）和一种故障（网络分区，节点仍处于活动状态但相互断开），而没有考虑网络延迟、节点失败或其他需要折中的情况。
> 因此，尽管CAP在历史上具有重大的影响力，但对于一个具体的系统设计来说，它可能没有太大的实际价值。

### 线性一致性
略（系统看起来好像只有一个数据副本，且所有的操作都是原子的）

所以当人们讨论到一致性的时候，最好在加入讨论前确定讨论的是什么样的一致性。

## MySQL

### Binary Log
MySQL的binlog主要用于备份，有三种格式
```text
show variables like 'binlog_format'
```

#### row-based logging
记录受影响的行

##### RBL不记录select或show
执行
```text
INSERT INTO binlog.test_01 (ID,`NO`)
VALUES (1,'1');
```
查看
```shell
/usr/local/mysql-8.0.27-macos11-arm64/bin/mysqlbinlog /usr/local/mysql/data/binlog.000006
```
或sql执行
```text
show binlog events in 'binlog.000006';
```
可得到
```text
# at 1080
#220917 20:22:55 server id 1  end_log_pos 1126 CRC32 0x333f33ae 	Write_rows: table id 100 flags: STMT_END_F

BINLOG '
H7wlYxMBAAAAPwAAADgEAAAAAGQAAAAAAAEABmJpbmxvZwAHdGVzdF8wMQACCA8CKAAAAQEAAgP8
/wAFqkAW
H7wlYx4BAAAALgAAAGYEAAAAAGQAAAAAAAEAAgAC/wABAAAAAAAAAAExrjM/Mw==
'/*!*/;
```
执行
```text
SELECT * FROM binlog.test_01;
```
无新增日志

##### RBL不记录无影响的update等
执行
```text
UPDATE binlog.test_01 SET `NO` = 2 WHERE ID = 2;
```
无新增日志，因为不存在ID = 2的记录

#### statement-based logging
记录运行的SQL，且可以在数据库运行时修改这个配置（有一些限制），运行时修改需要考虑的方面太多，最好一开始就定好，最近的版本默认使用row格式的。
```text
# at 1627
# at 1659
#220918 22:59:24 server id 1  end_log_pos 1659 CRC32 0xa967300e 	Intvar
SET INSERT_ID=2/*!*/;
#220918 22:59:24 server id 1  end_log_pos 1842 CRC32 0x957eaf9b 	Query	thread_id=18	exec_time=0	error_code=0
SET TIMESTAMP=1663513164/*!*/;
/* ApplicationName=DBeaver 21.3.2 - SQLEditor <Script.sql> */ INSERT INTO binlog.test_01 (`NO`)
VALUES ('2')
/*!*/;
```
与statement格式相同，不记录select。但会记录潜在有影响的sql
```text
# at 2041
#220918 23:14:19 server id 1  end_log_pos 2234 CRC32 0xe74b8799 	Query	thread_id=18	exec_time=0	error_code=0
SET TIMESTAMP=1663514059/*!*/;
/* ApplicationName=DBeaver 21.3.2 - SQLEditor <Script.sql> */ UPDATE binlog.test_01 SET `NO` = 3 WHERE ID = 3
/*!*/;
```

#### mixed logging
混合row-based与statement-based

#### binlog一致性
sync_binlog变量表示多少commit同步一次binlog，如需高可靠应设为1。
> In earlier MySQL releases, there was a chance of inconsistency between the table content and binary log content if a crash occurred, even with sync_binlog set to 1. For example, if you are using InnoDB tables and the MySQL server processes a COMMIT statement, it writes many prepared transactions to the binary log in sequence, synchronizes the binary log, and then commits the transaction into InnoDB. If the server unexpectedly exited between those two operations, the transaction would be rolled back by InnoDB at restart but still exist in the binary log. Such an issue was resolved in previous releases by enabling InnoDB support for two-phase commit in XA transactions. In MySQL 8.0, InnoDB support for two-phase commit in XA transactions is always enabled.

### XA 2PC

#### 系统模型
- 事务管理器管理那些访问存储在本地站点中的事务（或子事务）的执行。这个事务可以只在本地执行，也可以是全局事务在本站点的部分。
- 事务协调器协调该站点发起的各个事务的执行。

#### 提交协议
- 阶段1，协调器将<prepare T>加到日志，然后将prepare消息发送到执行事务T的所有站点上。
管理器确定后将<ready T>或<no T>加到日志，然后发送到协调器。
- 阶段2，协调器收到所有站点<ready T>或任一站点<no T>的响应，将<commit T>或<abort T>加到日志，并向所有站点发送结果。

#### 不足
- 单点。如果协调者不支持数据复制，而是在单节点上运行，那么它就是整个系统的单点故障（因为它的故障导致了很多应用阻塞在停顿事务所持有的锁上）。
而现实情况是，有许多协调者的实现默认情况下井非高可用，或者只支持最基本的复制。
- 许多服务器端应用程序都倾向于无状态模式，而所有的持久状态都保存在数据库中，这样应用服务器可以轻松地添加或删除实例（弹性）。
当协调者就是应用服务器的一部分时，协调者的日志成为可靠系统的重要组成部分，它要求与数据库本身一样重要（需要协调者日志恢复那些有疑问的事务）。
这样的应用服务器已经不再是无状态。

### Redo & Undo Log

#### 原子性与持久性
[数据库故障恢复机制的前世今生](http://catkang.github.io/2019/01/16/crash-recovery.html)

数据库为了实现原子性、持久性，需要先将更新内容记录到WAL（Write Ahead Log），如
- <T, X, V>：事务T写数据项X前其值为V（即Undo-Only）
- <T,commit>：事务T提交。

同时，数据库中数据的需要当前值以进行其它控制，如索引、加锁等。

原子性要求写回顺序是更新内容Log->Data->Commit Log。
由于非易失性的存储是以页、块与内存交换的，块在内存中被修改时里面的数据项存在三种状态：已修改已提交、已修改未提交、未修改，一个事务提交后（日志<T commit>写回）如果强制将内存当前值写回，会将已修改未提交的数据项也被写回，破坏了WAL。
因而MySQL/InnoDB结合了Redo与Undo Log。

#### 隔离性
MySQL/InnoDB复用了Undo Log来实现MVCC。

MVCC效果就像是在事务开始的时候对整个DB生成了一个快照，之后该事务的所有读请求都从这个快照上获取。
实现上因为时间空间复杂度，不会为每个事务都生成快照。
InnoDB的做法，是在事务第一次读取的时候获取一份ReadView，其中记录所有当前活跃的事务ID。
由于事务的ID是自增的，这样通过ReadView我们就可以知道在事务第一次读取时，哪些事务已经提交哪些还在运行。

作为存储历史版本的Undo Log，记录了每个版本的trx_id，对应的主索引的Record上也有这个值。
当一个事务拿着自己的ReadView访问某个表索引上的记录时，比较Record上的trx_id确定是否是当前事务可见的版本，如果不可见就沿着Record或Undo Log中记录的rollptr一路找更老的历史版本。

## 分布式
当谈论到一致性的时候，一般情况下是指事务的一致性。
传统的数据库（特别是关系型数据库）通过数据库的AID和应用程序员的正确编码来实现事务一致性。
但到了分布式时代，数据库（刚性）事务（XA 2PC）因并发等原因，已经越来越不受青睐（如MySQL作为Resource Manager）。
取而代之的是使用柔性事务来实现事务的一致性。
Seata是一种相关的框架。

### 柔性事务
在DDD中，聚合被用来定义一致性边界，不同聚合之间可以不满足一致性约束。

实现柔性事务的几种模式：
- Saga 事务拆分成多个子事务，事务失败时启动一系列具有相反效果的补偿子事务。
- TCC 服务化的2PC。

#### Saga
银行业常用，补偿事务通常称为冲正。
Saga又可以分成两种模式:
- Choreography
- Orchestration

Choreography模式各参与者在完成本地事务时发出事件。

Orchestration模式由一中央控制器控制Saga参与者执行本地事务。中央控制器的状态存放在本地的数据库中。

Saga理论由Hector & Kenneth 1987发表。

#### TCC
TCC模式需要在应用层实现Try\Confirm\Cancel接口。
相较于Saga，多了Try的操作（更复杂），能实现更好的隔离性。

不管Saga模式还是TCC模式，都需要允许命令/事件的乱序和幂等。