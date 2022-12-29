# 数据库冷知识

## 并发协议
|协议|事务间冲突时|死锁|需维护的时间戳与版本|饿死|级联回滚
|:---:|:---:|:---:|:---:|:---:|:---:
|两阶段封锁协议 two-phase locking|阻塞|无或回滚（取决于具体的机制）|事务的时间戳（取决于死锁机制），事务重启时间戳不变|按队列授予锁时无|提交后释放锁时无
|时间戳排序 timestamp-ordering|回滚，存在饿死时阻塞|无|读取该数据最大事务时间戳与写时间戳，事务重启使用新的时间戳|存在|存在
|有效性检查 validation|终止，存在饿死时阻塞|无|事务开始时间戳、开始检查时间戳、提交时间戳|存在|无
|多版本时间戳排序 multiversion timestamp-ordering|回滚|无|事务开始的时间戳，版本序列与读取各版本最大的事务时间戳与写时间戳|存在|存在

## JDBC
- JDBC属于动态SQL，Java嵌入式SQL叫SQLJ。
- 结果集比较复杂，见Oracle的Database JDBC Developer's Guide and Reference之Result Set。

## SQL
- 集合运算：union、intersect、except，保留重复在后面加all
- 集合比较：some、all
- 相关子查询：
```
select course_id
from section as S
where semester = 'Fall' and year = 2009 and
exists (select *
from section as T
where semester = 'Spring' and year = 2010 and
S.course_id = T.course_id);
```
- from子查询：
```
select name, salary, avg_salary
from instructor I1, lateral (select avg(salary) as avg_salary
from instructor I2
where I2.dept_name = I1.dept_name);
```
- 标量子查询，下面的语句可以检索出客户的订单数量，子查询语句会对第一个查询检索出的每个客户执行一次：
```
select cust_name, (select count(*)
from Orders
where Orders.cust_id = Customers.cust_id)
as orders_num
from Customers
order by cust_name;
```

## 隔离级别与并发协议
关于隔离级别，可以从其发展的历史来学习
1. 1992年ANSI提出了SQL-92，系统定义了四种隔离级别及三种phenomena（脏读、不可重复读、幻读）；
2. 1995年《A Critique of ANSI SQL Isolation Levels》指出了SQL-92中定义的不足，且不能做到与实现无关，并定义了快照隔离、游标稳定隔离级别和脏写、更新丢失、写倾斜的phenomena；
3. 1999年《Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions》严谨定义了可重复读。