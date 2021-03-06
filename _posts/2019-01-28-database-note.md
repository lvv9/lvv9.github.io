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
- 不管动态SQL（包括可滚动的、不可滚动的结果集）、嵌入式SQL，查询语句都有一个类似“游标”的概念，并且在交互过程中必须与数据库保持连接（不可滚动的结果集只有一个移动方向）。
- 结果集比较复杂，见Oracle的Database JDBC Developer's Guide and Reference之Result Set。
因为一个Statement默认是Auto-Commit的，只有SELECT FOR UPDATE会保持写锁，其它不加锁的情况（Auto-Commit）可能都会遇到并发问题。
所以并发的情况下需要把Auto-Commit关闭，用其他方式管理事务。

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

## Spring事务管理
