# 数据库冷知识


协议|事务间冲突时|时间戳与版本
:--:|:--:|:--:
两阶段封锁协议 two-phase locking|阻塞，发生死锁时可能回滚（取决于具体的机制）|
时间戳排序 timestamp-ordering|回滚，存在饿死时阻塞|事务的时间戳，一个数据版本，数据读时间戳与写时间戳|
有效性检查 validation|
多版本两阶段封锁 multiversion two-phase locking|
