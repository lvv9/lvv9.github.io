# 数据库冷知识

## 并发协议
协议|事务间冲突时|死锁|时间戳与版本
:--:|:--:|:--:|:--:
两阶段封锁协议 two-phase locking|阻塞|无或回滚（取决于具体的机制）|需维护事务的时间戳（取决于死锁机制），事务重启时间戳不变
时间戳排序 timestamp-ordering|回滚，存在饿死时阻塞|无|读取该数据最大事务时间戳与写时间戳，事务重启使用新的时间戳
有效性检查 validation|终止，存在饿死时阻塞|无|事务时间戳、完成检查时间戳、提交时间戳
多版本时间戳排序 multiversion timestamp-ordering|回滚|无|事务开始的时间戳，版本序列与读取各版本最大的事务时间戳与写时间戳
多版本两阶段封锁 multiversion two-phase locking|同两阶段封锁（读事务同多版本时间戳排序）|同两阶段封锁|事务开始的时间戳，版本序列与读取各版本最大的事务时间戳与写时间戳

## JDBC
* JDBC属于动态SQL，Java嵌入式SQL叫SQLJ。
* 不管动态SQL（包括可滚动的、不可滚动的结果集）、嵌入式SQL，查询语句都有一个类似“游标”的概念，并且在交互过程中必须与数据库保持连接（不可滚动的结果集只有一个移动方向）。
* 关于结果集的refreshRow和fetchSize：<https://stackoverflow.com/questions/2091659/behaviour-of-resultset-type-scroll-sensitive>
