# 数据库冷知识

协议|事务间冲突时|死锁|时间戳与版本
:--:|:--:|:--:|:--:
两阶段封锁协议 two-phase locking|阻塞|无或回滚（取决于具体的机制）|需维护事务的时间戳（取决于死锁机制），事务重启时间戳不变
时间戳排序 timestamp-ordering|回滚，存在饿死时阻塞|无|读取该数据最大事务时间戳与写时间戳，事务重启使用新的时间戳
有效性检查 validation|终止，存在饿死时阻塞|无|事务时间戳、完成检查时间戳、提交时间戳
多版本时间戳排序 multiversion timestamp-ordering|
多版本两阶段封锁 multiversion two-phase locking|
