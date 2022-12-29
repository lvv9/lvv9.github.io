# 分布式及ZooKeeper

## 《Designing Data-Intensive Application》读后
现在的服务端应用开发，越来越倾向于将服务端应用拆分成多个不同的微服务，同一服务也会以集群的方式部署。
这种应用组织形式，具有以下优势：
- 高可靠
集群中的某台机器出了问题，运行相同服务的平行的机器依然可以提供服务；某一服务集群都出现故障，仅影响部分功能；
- 可扩展
由于服务集群一般都是不固定机器数量的，因此可以随着负载的增加而扩展集群资源；
- 可维护
服务独立自治，一个团队在小型且易于理解（内聚）的环境中负责自己的服务，符合单一职责的设计原则。

但是随着应用被拆分成多个不同的服务部署在不同的机器，也带来了一些问题。
原本的由单一实体控制的服务间的可靠通信，变成了多个不同实体的不可靠通信。
解决这些问题，确实需要下不小的功夫。

在我们进一步分析系统之前，我们可以先建立一个分布式系统模型：部分同步、崩溃-恢复模型。同时，我们假定不同的机器只是简单的数据复制来同时对外提供服务。
- 部分同步，即大多数情况下网络延迟、进程暂停和时钟误差在合理的范围内；
- 崩溃恢复，节点可能会在任何时候发生崩惯，且可能会在一段（未知的）时间之后得到恢复并再次响应。

而我们的目标，是"最正确"的强一致，同时要求系统具备一定的容错能力。
强一致，就是线性一致，形式化的定义暂不讨论，基本思想就是让系统看起来好像只有一个数据副本，且所有的操作都是原子的。
共识算法加上其它一些机制，可以实现线性一致。

看起来好像只有一个数据副本，一个比较直观同时支持容错的做法就是，让某一个副本成为主副本，所有操作都按照主副本的顺序进行。
主副本故障则其它某一个副本代替，而主副本的选取，则通过共识算法选出。

## ZooKeeper小测
ZooKeeper便是这样做的，在满足大多数节点正常的条件下，系统可以正常提供服务：<br>
docker配置2台zookeeper的集群
```
version: '3.1'

services:
  zoo11:
    image: zookeeper
    restart: always
    hostname: zoo11
    ports:
      - 2191:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo11:2888:3888;2181 server.2=zoo12:2888:3888;2181

  zoo12:
    image: zookeeper
    restart: always
    hostname: zoo12
    ports:
      - 2192:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo11:2888:3888;2181 server.2=zoo12:2888:3888;2181
```
启动zoo11，不满足条件，日志显示
```
2022-02-13 09:26:06,211 [myid:1] - WARN  [QuorumConnectionThread-[myid=1]-4:QuorumCnxManager@401] - Cannot open channel to 2 at election address zoo12:3888
java.net.UnknownHostException: zoo12
at java.base/java.net.AbstractPlainSocketImpl.connect(Unknown Source)
at java.base/java.net.SocksSocketImpl.connect(Unknown Source)
at java.base/java.net.Socket.connect(Unknown Source)
at org.apache.zookeeper.server.quorum.QuorumCnxManager.initiateConnection(QuorumCnxManager.java:384)
at org.apache.zookeeper.server.quorum.QuorumCnxManager$QuorumConnectionReqThread.run(QuorumCnxManager.java:458)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
at java.base/java.lang.Thread.run(Unknown Source)
```
启动zoo12，满足条件
```
2022-02-13 09:27:01,429 [myid:1] - INFO  [QuorumPeer[myid=1](plain=0.0.0.0:2181)(secure=disabled):QuorumPeer@915] - Peer state changed: following - broadcast
```
```
2022-02-13 09:27:01,429 [myid:2] - INFO  [QuorumPeer[myid=2](plain=0.0.0.0:2181)(secure=disabled):QuorumPeer@915] - Peer state changed: leading - broadcast
```
docker配置3台zookeeper的集群并启动zoo1、zoo2（不启动zoo3），选出zoo2 leading<br>
启动zoo3，zoo3 following
```
2022-02-13 09:35:29,086 [myid:3] - INFO  [QuorumPeer[myid=3](plain=0.0.0.0:2181)(secure=disabled):QuorumPeer@915] - Peer state changed: following - broadcast
```
在之前zoo11、zoo12集群的基础上增加zoo13
```
2022-02-13 09:52:41,058 [myid:3] - INFO  [QuorumPeer[myid=3](plain=0.0.0.0:2181)(secure=disabled):QuorumPeer@915] - Peer state changed: following - broadcast
```
再增加zoo14、zoo15（配置里的ZOO_SERVERS仅包括zoo11、zoo12、zoo1x）并启动后再停掉zoo11、zoo12，虽然动态地看3/5满足大多数节点运行的条件，但停掉后zoo13、zoo14、zoo15都不能服务
```
2022-02-13 10:07:54,903 [myid:5] - WARN  [QuorumConnectionThread-[myid=5]-8:QuorumCnxManager@401] - Cannot open channel to 1 at election address zoo11/172.18.0.2:3888
java.net.NoRouteToHostException: No route to host (Host unreachable)
at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
at java.base/java.net.AbstractPlainSocketImpl.doConnect(Unknown Source)
at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(Unknown Source)
at java.base/java.net.AbstractPlainSocketImpl.connect(Unknown Source)
at java.base/java.net.SocksSocketImpl.connect(Unknown Source)
at java.base/java.net.Socket.connect(Unknown Source)
at org.apache.zookeeper.server.quorum.QuorumCnxManager.initiateConnection(QuorumCnxManager.java:384)
at org.apache.zookeeper.server.quorum.QuorumCnxManager$QuorumConnectionReqThread.run(QuorumCnxManager.java:458)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
at java.base/java.lang.Thread.run(Unknown Source)
2022-02-13 10:07:54,903 [myid:5] - WARN  [QuorumConnectionThread-[myid=5]-7:QuorumCnxManager@401] - Cannot open channel to 2 at election address zoo12/172.18.0.3:3888
java.net.NoRouteToHostException: No route to host (Host unreachable)
at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
at java.base/java.net.AbstractPlainSocketImpl.doConnect(Unknown Source)
at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(Unknown Source)
at java.base/java.net.AbstractPlainSocketImpl.connect(Unknown Source)
at java.base/java.net.SocksSocketImpl.connect(Unknown Source)
at java.base/java.net.Socket.connect(Unknown Source)
at org.apache.zookeeper.server.quorum.QuorumCnxManager.initiateConnection(QuorumCnxManager.java:384)
at org.apache.zookeeper.server.quorum.QuorumCnxManager$QuorumConnectionReqThread.run(QuorumCnxManager.java:458)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
at java.base/java.lang.Thread.run(Unknown Source)
```
启动zoo11、zoo12，停掉zoo13、zoo14、zoo15，集群正常服务
```
# ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: leader
```
再停掉zoo11，集群停止服务
```
# ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Error contacting service. It is probably not running.
```
从以上案例可以看出，zookeeper集群大多数节点正常时才提供服务。
以及，每台机器运行所需要的集群的信息，是偏静态的，即配置在ZOO_SERVERS中的集群信息。

## 分布式锁
首先，什么是分布式锁？究竟这意味着锁服务是分布式的？还是锁服务是管理分布式系统的锁请求？
在初步找了几个实现方法后，这里取后者描述情况，即分布式锁服务可以是单点的。

另外一个实现方面的问题是，一般的锁管理，锁管理器、锁请求方和资源都是在一起的，可以较可靠地保证资源的互斥访问。
但是分布式锁服务，锁管理器、锁请求方与资源可能是分开的各自独立的集群。
在锁存在租期的情况下，因为锁请求方的时钟不会与锁管理服务同步，锁请求方会遭遇进程暂停、网络延迟，会导致超时已被释放的锁破坏了资源。
因此，在这种情况下，需要注意使用fencing token，保证资源本身能提供机制进行保护，可见《DDIA》第八章。
token可以是随机的，这样资源自身的保护在CAS比较时需使用request token == resource token判断；token也可以是递增的，条件则可以放宽到request token >= resource token。

### 数据库行锁
既然可以是单点的，尝试比较简单可靠的实现，锁管理器与资源都由同一实体管理——数据库行锁。
之前参与过的一些项目，见（ [MySQL并发实验](https://liuweiqiang.me/2021/12/03/mysql-concurrent-control-test.html) )，就是利用数据库业务对象的行锁（select for update），来控制并发的。
当然，这种锁依赖了业务对象，可以用另外的表（列可包含命名空间+业务ID）来泛化锁。

这种方法会产生较多的长事务，而且锁管理与资源保护是耦合在一起的，使用场景比较有限。

### 数据库唯一约束
另一个比较简单的基于数据库的实现——唯一索引。
个人实验性的实现，可见（ [唯一索引分布式锁](https://github.com/lvv9/distributed-locking) ）

### ZooKeeper实现
利用ZooKeeper，我们可以比较快速地实现有容错能力的分布式锁管理。
同样的实验性实现，见同一项目（ [ZooKeeper分布式锁](https://github.com/lvv9/distributed-locking) ）