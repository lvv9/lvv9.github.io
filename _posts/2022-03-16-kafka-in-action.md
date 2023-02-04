# kafka部署验证
kafka的配置有很多，这里挑一些比较感兴趣的验证一下，官方说明见 [这里](https://kafka.apache.org/31/documentation.html#brokerconfigs)

## 初始部署配置
```properties
broker.id=0
listeners=PLAINTEXT://0.0.0.0:9092 #监听所有网卡的9092
advertised.listeners=PLAINTEXT://kafka:9092 #注册在zookeeper上的信息
zookeeper.connect=zoo1:2181,zoo2:2181,zoo3:2181 #zookeeper集群信息
log.dirs=/tmp/kafka-logs
```
由于示例中的kafka是部署在docker中的，且连接了多个网络（即多个网卡），所以监听0.0.0.0
```shell
docker container inspect kafka
```
```text
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "e3554b6f538add748eada670007eaf2bd76de26b944e8d3db0f99a8cf2a43435",
                    "EndpointID": "ceac7cc287def9c2241be645e0b7b739d907d645ff5e607428bcebccbea89ebe",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                },
                "docker_default": {
                    "IPAMConfig": {},
                    "Links": null,
                    "Aliases": [
                        "f7195078ce1b",
                        "kafka"
                    ],
                    "NetworkID": "b127a7674eb41468039d4d181b40397ccf710755594c591adb528d51cb558a90",
                    "EndpointID": "0f6ef9aab765ff783818da7ee2bb92c432ccaf9eb2337b2a2bff77b369eb6211",
                    "Gateway": "172.18.0.1",
                    "IPAddress": "172.18.0.5",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:12:00:05",
                    "DriverOpts": {}
                }
            }
```
```shell
ifconfig
```
```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 95619  bytes 140011940 (140.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 21005  bytes 1171563 (1.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.5  netmask 255.255.0.0  broadcast 172.18.255.255
        ether 02:42:ac:12:00:05  txqueuelen 0  (Ethernet)
        RX packets 272  bytes 25462 (25.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 441  bytes 34869 (34.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 28  bytes 1954 (1.9 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 28  bytes 1954 (1.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
启动并查看
```shell
apt install screen net-tools
screen -s /bin/bash -S kafka
bin/kafka-server-start.sh config/server.properties #ctrl + a + d to detach
netstat -tunlp
```
```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:9092            0.0.0.0:*               LISTEN      5762/java
```
zookeeper zkCli.sh执行
```shell
get /brokers/ids/0 
```
```text
{"features":{},"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://kafka:9092"],"jmx_port":-1,"port":9092,"host":"kafka","version":5,"timestamp":"1647406227247"}
```
如果把advertised.listeners注释掉重启，由于0.0.0.0不能被其它机器解释，kafka会在启动时异常
```text
[2022-03-16 14:54:18,511] ERROR Exiting Kafka due to fatal exception (kafka.Kafka$)
java.lang.IllegalArgumentException: requirement failed: advertised.listeners cannot use the nonroutable meta-address 0.0.0.0. Use a routable IP address.
        at scala.Predef$.require(Predef.scala:337)
        at kafka.server.KafkaConfig.validateValues(KafkaConfig.scala:2136)
        at kafka.server.KafkaConfig.<init>(KafkaConfig.scala:1992)
        at kafka.server.KafkaConfig.<init>(KafkaConfig.scala:1466)
        at kafka.Kafka$.buildServer(Kafka.scala:67)
        at kafka.Kafka$.main(Kafka.scala:87)
        at kafka.Kafka.main(Kafka.scala)
```
修改listeners，重启
```properties
listeners=PLAINTEXT://kafka:9092
```
```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 172.17.0.2:9092         0.0.0.0:*               LISTEN      7591/java
```
这里比较奇怪，因为用nslookup查询出来是不一样的结果
```shell
apt install dnsutils
nslookup kafka
```
```text
Server:		127.0.0.11
Address:	127.0.0.11#53

Non-authoritative answer:
Name:	kafka
Address: 172.18.0.5
```
理所当然zoo1上查询失败
```text
root@zoo1:~/kafka_2.13-3.1.0# bin/kafka-configs.sh --bootstrap-server kafka:9092 --entity-type brokers --entity-name 0 --describe
Dynamic configs for broker 0 are:
[2022-03-16 07:13:39,000] WARN [AdminClient clientId=adminclient-1] Connection to node -1 (kafka/172.18.0.5:9092) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
```
另外，kafka还支持动态配置
```shell
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-name 0 --alter --delete-config advertised.listeners
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-name 0 --alter --add-config advertised.listeners=PLAINTEXT://172.18.0.5:9092
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-name 0 --describe
```
```text
Dynamic configs for broker 0 are:
  advertised.listeners=PLAINTEXT://172.18.0.5:9092 sensitive=false synonyms={DYNAMIC_BROKER_CONFIG:advertised.listeners=PLAINTEXT://172.18.0.5:9092, STATIC_BROKER_CONFIG:advertised.listeners=PLAINTEXT://kafka:9092}
```
zookeeper上查看
```text
[zk: localhost:2181(CONNECTED) 1] get /brokers/ids/0 
{"features":{},"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://172.18.0.5:9092"],"jmx_port":-1,"port":9092,"host":"172.18.0.5","version":5,"timestamp":"1647415769434"}
```
以下是官方的说明
```markdown
- read-only: Requires a broker restart for update
- per-broker: May be updated dynamically for each broker
- cluster-wide: May be updated dynamically as a cluster-wide default. May also be updated as a per-broker value for testing.
All configs that are configurable at cluster level may also be configured at per-broker level (e.g. for testing). If a config value is defined at different levels, the following order of precedence is used:
- Dynamic per-broker config stored in ZooKeeper
- Dynamic cluster-wide default config stored in ZooKeeper
- Static broker config from server.properties
- Kafka default, see broker configs
```

## 控制器
```text
controller.*
metadata.*
node.id
process.roles
```
略，zookeeper版kafka不太需要。

## 消息日志
因为消息基本上是final的，kafka使用日志来存储消息，类似于MySQL的binlog，只不过MySQL除了binlog还有其它的结构。<br>
3.1.0版的开箱配置有个
```properties
log.retention.hours=168
```
可配置的还包括
```text
log.retention.minutes
log.retention.ms
```
涉及到kafka的消息日志，如果配置为可以删除10s前的日志
```shell
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-name 0 --alter --add-config log.retention.ms=10000
bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092 #ctrl + c to stop
```
查看
```shell
bin/kafka-dump-log.sh --files /tmp/kafka-logs/quickstart-events-0/00000000000000000000.log --print-data-log
```
而后log文件被删除，增加了后缀为.deleted的文件，最后也被删除。<br>
文件夹后缀代表分区，文件名代表第一条消息的编号。

## 副本
```text
default.replication.factor
```
副本数
```text
min.insync.replicas
```
最少同步副本数，需与producer配置acks配合使用，以保证消息的durability。如果创建一个3副本主题
```shell
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-topic --partitions 2 --replication-factor 3
```
可以在其它节点（示例为kafka3）查看到
```text
root@kafkf3:/# ls /tmp/kafka-logs/
cleaner-offset-checkpoint    my-topic-1
log-start-offset-checkpoint  recovery-point-offset-checkpoint
meta.properties              replication-offset-checkpoint
my-topic-0
```
修改为最少2，然后发送消息
```shell
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-default --alter --add-config min.insync.replicas=2
bin/kafka-console-producer.sh --topic my-topic --producer-property acks=all --bootstrap-server localhost:9092 #ctrl + c to stop
```
```text
>replica
>replica2
>[2022-03-18 01:34:08,313] WARN [Producer clientId=console-producer] Got error produce response with correlation id 7 on topic-partition my-topic-0, retrying (2 attempts left). Error: NOT_ENOUGH_REPLICAS (org.apache.kafka.clients.producer.internals.Sender)
[2022-03-18 01:34:08,422] WARN [Producer clientId=console-producer] Got error produce response with correlation id 8 on topic-partition my-topic-0, retrying (1 attempts left). Error: NOT_ENOUGH_REPLICAS (org.apache.kafka.clients.producer.internals.Sender)
[2022-03-18 01:34:08,528] WARN [Producer clientId=console-producer] Got error produce response with correlation id 9 on topic-partition my-topic-0, retrying (0 attempts left). Error: NOT_ENOUGH_REPLICAS (org.apache.kafka.clients.producer.internals.Sender)
[2022-03-18 01:34:08,638] ERROR Error when sending message to topic my-topic with key: null, value: 8 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
org.apache.kafka.common.errors.NotEnoughReplicasException: Messages are rejected since there are fewer in-sync replicas than required.
```
发送replica消息时停止了kafka3，发送replica2消息时停止了kafka2。同时，kafka日志中仅存在replica的消息
```text
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-1/00000000000000000000.log --print-data-log
Dumping /tmp/kafka-logs/my-topic-1/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false isControl: false position: 0 CreateTime: 1647538318648 size: 75 magic: 2 compresscodec: none crc: 684786291 isvalid: true
| offset: 0 CreateTime: 1647538318648 keySize: -1 valueSize: 7 sequence: -1 headerKeys: [] payload: replica
```
如果producer设置为acks=1
```text
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-console-producer.sh --topic my-topic --producer-property acks=1 --bootstrap-server localhost:9092
>replica3
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-0/00000000000000000000.log --print-data-log
Dumping /tmp/kafka-logs/my-topic-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 2 isTransactional: false isControl: false position: 0 CreateTime: 1647539123622 size: 76 magic: 2 compresscodec: none crc: 2948014261 isvalid: true
| offset: 0 CreateTime: 1647539123622 keySize: -1 valueSize: 8 sequence: -1 headerKeys: [] payload: replica3
```

## 分区
```text
num.partitions
```
上面已在创建主题时指定分区数。
```text
auto.leader.rebalance.enable
leader.imbalance.check.interval.seconds
leader.imbalance.per.broker.percentage
```
分区最主要的目的是平衡负载，但在运行过程中可能因为扩展性、故障等而增加、减少分区。<br>
比较常见的做法是像Redis这样，一开始就创建远超实际节点数的分区，且分区数固定，然后为每个节点动态地分配分区。访问路由表就是分区到节点的映射。<br>
Kafka分区比较灵活，分区数可以由用户在创建主题时指定，并且可以在运行过程中增加分区数量（不影响旧的消息），并且根据以上3个配置项进行再平衡。<br>
再平衡与路由可能涉及共识问题，富有挑战性，这里不再深入。

## 消费者组
不同的消费者组通常需要对消息做不同的处理。比如，A组发短信，B组发邮件。<br>
因此一个消费者组需要订阅一个主题的所有消息。这种多个消费者组的工作模式，叫扇出式（参考DDIA）。<br>
而一个分区只能被一个消费者组内的一个消费者消费，也就是说消费者组中的消费者数量大于分区数的话，会存在某些消费者不干活。这种模式，叫负载均衡式。<br>
这样的话，同一分区的消息，会按照分区内的顺序被消费。
```text
offsets.retention.minutes
offsets.topic.num.partitions
offsets.topic.replication.factor
```
kafka有个内部的topic【__consumer_offsets】，用来记录消费者消费了哪些消息
```text
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
__consumer_offsets
my-topic
quickstart-events
bin/kafka-console-consumer.sh --topic quickstart-events --bootstrap-server localhost:9092 --group test_group
```
```text
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-consumer-groups.sh --describe --group test_group --bootstrap-server localhost:9092

GROUP           TOPIC             PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID
test_group      quickstart-events 0          9               9               0               console-consumer-54931ba4-ba57-4dcb-badc-a11bf87e6b2f /172.20.0.2     console-consumer
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-consumer-groups.sh --reset-offsets --group test_group --bootstrap-server localhost:9092 --execute --topic quickstart-events --to-earliest

GROUP                          TOPIC                          PARTITION  NEW-OFFSET     
test_group                     quickstart-events              0          2              
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-consumer-groups.sh --describe --group test_group --bootstrap-server localhost:9092

Consumer group 'test_group' has no active members.

GROUP           TOPIC             PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
test_group      quickstart-events 0          2               9               7               -               -               -
```
再次运行上面的消费者命令，可以看到之前的消息。<br>
这里有个小插曲，因为上面设置了min.insync.replicas=2，在使用console-consumer时一直消费不了消息，删除后就可以了。<br>

## 其它高可用、高可靠配置
```text
unclean.leader.election.enable
```
是否允许非ISR选举为分区leader，默认为false。为方便演示，新建2副本主题
```shell
bin/kafka-topics.sh --create --topic two-replica-topic --bootstrap-server localhost:9092 --config unclean.leader.election.enable=true --replication-factor 2
```
查看leader
```text
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-topics.sh --describe --bootstrap-server localhost:9092,kafkf3:9092 --topic two-replica-topic
Topic: two-replica-topic	TopicId: nwYIqUb2SBi_kiF14n35pg	PartitionCount: 1	ReplicationFactor: 2	Configs: segment.bytes=1073741824,unclean.leader.election.enable=true
	Topic: two-replica-topic	Partition: 0	Leader: 2	Replicas: 2,1	Isr: 2,1
```
停止1号机
```text
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-topics.sh --describe --bootstrap-server localhost:9092,kafkf3:9092 --topic two-replica-topic
Topic: two-replica-topic	TopicId: nwYIqUb2SBi_kiF14n35pg	PartitionCount: 1	ReplicationFactor: 2	Configs: segment.bytes=1073741824,unclean.leader.election.enable=true
	Topic: two-replica-topic	Partition: 0	Leader: 2	Replicas: 2,1	Isr: 2
```
启动消费者，生产者，生产消费1条消息，停止2号机，启动1号机
```text
root@kafka:~/kafka_2.13-3.1.0# bin/kafka-topics.sh --describe --bootstrap-server localhost:9092,kafkf3:9092 --topic two-replica-topic
Topic: two-replica-topic	TopicId: nwYIqUb2SBi_kiF14n35pg	PartitionCount: 1	ReplicationFactor: 2	Configs: segment.bytes=1073741824,unclean.leader.election.enable=true
	Topic: two-replica-topic	Partition: 0	Leader: 1	Replicas: 2,1	Isr: 1
```
停止消费者，发布第2条消息，然后--from-beginning启动消费者，只能消费到第2条消息，再次启动2号机，查看日志
```text
root@kafkf3:/# ~/kafka_2.13-3.1.0/bin/kafka-dump-log.sh --print-data-log --files /tmp/kafka-logs/two-repli
ca-topic-0/00000000000000000000.log 
Dumping /tmp/kafka-logs/two-replica-topic-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 3 isTransactional: false isControl: false position: 0 CreateTime: 1648388037650 size: 69 magic: 2 compresscodec: none crc: 1785415631 isvalid: true
| offset: 0 CreateTime: 1648388037650 keySize: -1 valueSize: 1 sequence: -1 headerKeys: [] payload: b
```
```text
broker.rack
```
配置机架/机房，与分区、副本分配算法有关。

## ~~purgatory~~
~~*.purgatory.purge.interval.requests~~

## API
可以见 [demo项目](https://github.com/lvv9/kafka-demo) <br>
从这里也可以看出，producer API是异步的，如果在消息在被发送前producer挂掉，数据就会丢失。

## 消息顺序
如果生产者是集群，一般都不会要求按某种顺序生产消息（除非业务规则上的约束）。如果生产者是单机，或者要求同一机器的消息必须是顺序的，那么
1. 生产者生产的消息都需要发送到同一分区
2. 由于生产者API是异步的，需要在broker ack后也就是异步转同步解除阻塞后才发送下一消息

另外有几个producer配置与顺序有关
```text
retries
enable.idempotence
max.in.flight.requests.per.connection
```
按照官方文档的说明，似乎异步调用正常时消息也会按照send()调用的前后排序，以上配置是为了在异步异常时保证顺序，但最保险的还是在应用层有明确的offset返回值时才发送下一消息。<br>
在获取offset后，可以以offset作为消息的顺序。实现消费者严格按照顺序消费也面临同样的问题，值得进一步深入。不过一般情况下对顺序的要求不是很高。

因为实现消费者按序消费是一件代价很高的事情，个人认为应尽量避免。

### 事件顺序
在使用事件驱动这种模式时，有时不同业务事件的产生就已经有一定的顺序。
这种情况下，可以为每个事件附加一个时间戳，以此来当作递增的版本，消费者依据时间戳来保护资源，此时需要误差在一定的范围内。

除了上面的这种版本号保护机制（fail-fast），消费者还可以进行类似于TCP这样的处理，当发现前序事件未接收时将后发生先到达的事件缓存。