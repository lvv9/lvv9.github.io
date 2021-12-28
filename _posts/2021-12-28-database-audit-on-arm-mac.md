# MariaDB数据库审计插件
数据在现在的信息时代几乎是企业最重要的资产，企业级的应用在数据库方面有着非常高的要求。而审计日志对数据库各种操作进行记录，有着重要的作用。

MySQL社区版默认有个general_log，默认是关闭的，说是影响性能。
> SHOW GLOBAL VARIABLES LIKE 'general_log%';

虽说general_log不建议用，我们也还可以用一些第三方的插件，比如MariaDB Audit Plugin，来实现审计日志。

在机器换了最新的Apple Silicon笔记本的情况下，经过一段长时间的捣鼓MySQL安装MariaDB Audit Plugin，最后还是以失败告终：<br>
提取的ARM版MariaDB插件，安装提示莫名其妙的错误；Percona插件能安装，但是不起作用；McAfee插件无ARM版。

本来打算就此作罢，但最近因为接触了Docker，且Docker上有MariaDB的包，因此装上了MariaDB，审计功能能正常使用，不晓得ARM版MariaDB是否也能正常使用。<br>
按照 [Docker MariaDB](https://hub.docker.com/_/mariadb) 页面上的指引装好后，继续按照 [MariaDB Audit Plugin](https://mariadb.com/kb/en/mariadb-audit-plugin/) 上指引即可：
```
SHOW GLOBAL VARIABLES LIKE 'plugin_dir';
SHOW PLUGINS;
INSTALL SONAME 'server_audit';
SHOW GLOBAL VARIABLES LIKE 'server_audit%';
SET GLOBAL server_audit_logging = 'ON';
SET GLOBAL server_audit_events = 'CONNECT,QUERY,TABLE';
SHOW GLOBAL VARIABLES '%general_log%';
```