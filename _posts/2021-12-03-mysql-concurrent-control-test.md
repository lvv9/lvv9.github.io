# MySQL并发实验

在本人以前参与过的项目中，数据库用的是oracle，并发控制主要是靠数据库的select for update。而最近项目开发过程中，几乎都没有看到select for update的用法。于是乎，打算实际测试一下，避免在后面遇到这样那样的坑。

如果不使用select for update（mysql默认RR级别）来控制并发，一般都会使用乐观的方式更新记录，即
```
update table set name = 'lwq' where id = '1' and name = 'lwq_origin';
```
这种CAS操作，其实与select for update的用法差别不是很大，特别是对oracle这样的没有gap lock的数据库。因为一条记录在被update以后，就会被锁住，直至事务结束：
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

T2
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

T1
【阻塞超时前返回数据】
```
前面的例子，是在记录存在的情况下才成立的，在记录不存在的时候，情况有一些变化。
