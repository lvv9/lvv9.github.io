# Redis集群再平衡
https://redis.io/docs/management/scaling/
https://severalnines.com/blog/hash-slot-resharding-and-rebalancing-redis-cluster/

## 配置
docker compose file
```text
services:
  redis1:
    image: redis
    restart: always
    hostname: redis1
    ports:
      - 7000:7000
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - /Users/lwq/docker/redis/redis.conf:/usr/local/etc/redis/redis.conf
  ...
```
redis.conf
```text
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

## 初始化
为避免容器解析域名异常，以下命令参数使用ip
```text
Node XXX replied with error:
ERR Invalid node address specified: XXX
```
```shell
redis-cli --cluster create 172.18.0.6:7000 172.18.0.7:7000 172.18.0.8:7000 172.18.0.9:7000 172.18.0.10:7000 172.18.0.11:7000 --cluster-replicas 1
```

## 验证
重新调整集群（ https://redis.io/commands/cluster-nodes/ ）：
```text
# redis-cli -p 7000 cluster nodes
e38edf4a3dbe6079bf4b3582375196346511d6d4 172.18.0.10:7000@17000 slave 76636d51e59e664038332c0396257bb049bacbb2 0 1692541392000 1 connected
a566653e35493dbd160ac70cef266bbdea809cc5 172.18.0.13:7000@17000 slave da5d2379241fe06722371b2a28fb5f9c774a1cf1 0 1692541392391 2 connected
76636d51e59e664038332c0396257bb049bacbb2 172.18.0.11:7000@17000 myself,master - 0 1692541391000 1 connected 0-5460
4b1301f18426549ffb45af877877407e35d7aa55 172.18.0.2:7000@17000 master - 0 1692541391000 8 connected 10923-16383
da5d2379241fe06722371b2a28fb5f9c774a1cf1 172.18.0.12:7000@17000 master - 0 1692541392700 2 connected 5461-10922
```
增加一个节点
```text
# redis-cli --cluster add-node 172.18.0.4:7000 172.18.0.11:7000
```
重新平衡
```text
# redis-cli --cluster rebalance 172.18.0.11:7000 --cluster-use-empty-masters
# redis-cli -p 7000 cluster nodes
e38edf4a3dbe6079bf4b3582375196346511d6d4 172.18.0.10:7000@17000 slave 76636d51e59e664038332c0396257bb049bacbb2 0 1692541854000 1 connected
a566653e35493dbd160ac70cef266bbdea809cc5 172.18.0.13:7000@17000 slave da5d2379241fe06722371b2a28fb5f9c774a1cf1 0 1692541853503 2 connected
76636d51e59e664038332c0396257bb049bacbb2 172.18.0.11:7000@17000 myself,master - 0 1692541854000 1 connected 1365-5460
4b1301f18426549ffb45af877877407e35d7aa55 172.18.0.2:7000@17000 master - 0 1692541854528 8 connected 12288-16383
da5d2379241fe06722371b2a28fb5f9c774a1cf1 172.18.0.12:7000@17000 master - 0 1692541853094 2 connected 6827-10922
bbaad08d30349852672bcefa4e6b4350e600a702 172.18.0.4:7000@17000 master - 0 1692541853503 9 connected 0-1364 5461-6826 10923-12287
```
删除一个节点
> in order to remove a master node it must be empty.
```text
# redis-cli --cluster reshard 172.18.0.11:7000 --cluster-from bbaad08d30349852672bcefa4e6b4350e600a702 --cluster-to 76636d51e59e664038332c0396257bb049bacbb2 --cluster-slots 4095 --cluster-yes
```
重新平衡
```text
# redis-cli --cluster rebalance 172.18.0.11:7000
# redis-cli -p 7000 cluster nodes
e38edf4a3dbe6079bf4b3582375196346511d6d4 172.18.0.10:7000@17000 slave 76636d51e59e664038332c0396257bb049bacbb2 0 1692542856000 12 connected
a566653e35493dbd160ac70cef266bbdea809cc5 172.18.0.13:7000@17000 slave da5d2379241fe06722371b2a28fb5f9c774a1cf1 0 1692542857193 14 connected
76636d51e59e664038332c0396257bb049bacbb2 172.18.0.11:7000@17000 myself,master - 0 1692542856000 12 connected 2731-6826 10923-12287
4b1301f18426549ffb45af877877407e35d7aa55 172.18.0.2:7000@17000 master - 0 1692542856176 13 connected 0-1365 12288-16383
da5d2379241fe06722371b2a28fb5f9c774a1cf1 172.18.0.12:7000@17000 master - 0 1692542855666 14 connected 1366-2730 6827-1092
```