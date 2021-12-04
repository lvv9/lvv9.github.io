# MySQL并发实验

在本人以前参与过的项目中，数据库用的是Oracle，并发控制主要是靠数据库的select for update。而最近项目开发过程中，几乎都没有看到select for update的用法。于是乎，打算实际测试一下，避免在后面遇到这样那样的坑。

如果不使用select for update（mysql默认RR级别）来控制并发，一般都会使用乐观的方式更新记录，即
```
update table set name = 'lwq' where id = '1' and name = 'lwq_origin';
```
这种CAS操作，其实与select for update的用法差别不是很大，特别是对Oracle这样的没有gap lock的数据库。因为一条记录在被update以后，就会被锁住，直至事务结束：
```
T1
mysql> select * from test_for_update;
+-------------------+------+
| idtest_for_update | name |
+-------------------+------+
|                 1 | lwq  |
+-------------------+------+
1 row in set (0.00 sec)

T2
mysql> update test_for_update set name = 'lwq2' where idtest_for_update = '1';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

T1
mysql> select * from test_for_update where idtest_for_update = '1' for update;
【阻塞超时后】
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

T1
mysql> select * from test_for_update where idtest_for_update = '1' for update;
【阻塞超时前返回数据】

T2
mysql> commit;
Query OK, 0 rows affected (0.00 sec)
```
以上是记录存在的情况，在记录不存在的时候，情况类似：
```
T1
mysql> update test_for_update set name = 'lwq2' where idtest_for_update = '2';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0

T2
mysql> insert into test_for_update values ('3', 'lwq');
【阻塞直到T1结束】
Query OK, 1 row affected (12.67 sec)
我们知道，理想的锁并发控制（阻止幻象），应该只对where后面的谓词加锁。但是，MVCC的MySQL实现上基于性能上的考虑，扩大了锁的范围，使用gap lock来阻止非持有锁事务插入记录，从而阻止幻象。
```
T1执行select for update加锁同update。

基于以上实验，可以看出，select for update与CAS的update差别不大。但为什么却没有用select for update呢？
想来想去，select for update与update乐观地更新记录还是有这样的区别的：
一般乐观update前，会先不加锁select，如果数据不存在，不进行update而执行其它操作。而select for update在数据不存在的情况下一开始就会把gap锁住，造成后续insert的阻塞。

总的来说，在使用MySQL的事务时，最好用CAS的方式操作。而对于Oracle，因为没有gap lock，所以都是可以的。
