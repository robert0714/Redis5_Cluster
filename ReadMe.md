https://redis.io/topics/cluster-tutorial
## 關於 Redis Cluster
* 僅適合於 Redis3.0（包括3.0）以上版本  
* 每個 redis cluster 有 16384 個 hash slot，key 經過 CRC16 計算後決定儲存的 cluster node  
* 每個 redis node 都需要開放兩個 TCP port：一個是 redis client 連線使用的 port (常見為 6379) 另一個是 cluster bus 用來做 node 間的交互溝通 (固定為 client port 加上 10000)
* 僅支援單一 database  
* 不支援多個 key 的操作 (可能需要跨 node)  
* 最少需要三個 master node，但建議使用 3 個 master nodes + 3 個 slave node  

## Compile

```bash
curl -O http://download.redis.io/releases/redis-5.0.10.tar.gz
tar xzf redis-5.0.10.tar.gz
cd redis-5.0.10
make
```
## Configuration
basic
```conf
port 7000 # 每個 node 的 client port
cluster-enabled yes # 啟用 redis cluster
cluster-config-file nodes_7000.conf # 每個 node 需要獨立，cluster 自行維護使用，不需人為介入
cluster-node-timeout 5000 # node 判斷失效的時間
appendonly yes # 啟用 aof
```
advance
```conf
port 7000 # 每個 node 的 client port
cluster-enabled yes # 啟用 redis cluster
cluster-config-file nodes_7000.conf # 每個 node 需要獨立，cluster 自行維護使用，不需人為介入
cluster-node-timeout 5000 # node 判斷失效的時間
appendonly yes # 啟用 aof
daemonize yes # 背景執行
bind x.x.x.x # 允許 listen 特定 ip 的連線
requirepass password # 密碼設定
masterauth password # 從 master 的密碼
```
directory
```bash
├── 7000
│   └── redis.conf
├── 7001
│   └── redis.conf
├── 7002
│   └── redis.conf
├── 7003
│   └── redis.conf
├── 7004
│   └── redis.conf
├── 7005
│   └── redis.conf
```
## Startup
starting up redis instances
```bash
./redis-5.0.10/src/redis-server  ./7000/redis.conf
./redis-5.0.10/src/redis-server  ./7001/redis.conf
./redis-5.0.10/src/redis-server  ./7002/redis.conf
./redis-5.0.10/src/redis-server  ./7003/redis.conf
./redis-5.0.10/src/redis-server  ./7004/redis.conf
./redis-5.0.10/src/redis-server  ./7005/redis.conf

```
Joining the Cluster

```bash
$  ./redis-5.0.10/src/redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:7004 to 127.0.0.1:7000
Adding replica 127.0.0.1:7005 to 127.0.0.1:7001
Adding replica 127.0.0.1:7003 to 127.0.0.1:7002
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 09ed839f9c1c06f0b350426cf76a69793e9c97f3 127.0.0.1:7000
   slots:[0-5460] (5461 slots) master
M: be34587acf7132b70008fc78666a9e7d20ee7cfc 127.0.0.1:7001
   slots:[5461-10922] (5462 slots) master
M: 7d6334bebbca4386e775b7bbfdb29d758f537528 127.0.0.1:7002
   slots:[10923-16383] (5461 slots) master
S: 8a194deaa040bd97ac5fc425a13f394de91f5bcf 127.0.0.1:7003
   replicates be34587acf7132b70008fc78666a9e7d20ee7cfc
S: 65e44433530c8acbd81a301958f939bb42921820 127.0.0.1:7004
   replicates 7d6334bebbca4386e775b7bbfdb29d758f537528
S: 73645156ab1df9fd4fdc9ee2ee0b6750238381ec 127.0.0.1:7005
   replicates 09ed839f9c1c06f0b350426cf76a69793e9c97f3
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
..
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 09ed839f9c1c06f0b350426cf76a69793e9c97f3 127.0.0.1:7000
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 7d6334bebbca4386e775b7bbfdb29d758f537528 127.0.0.1:7002
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 8a194deaa040bd97ac5fc425a13f394de91f5bcf 127.0.0.1:7003
   slots: (0 slots) slave
   replicates be34587acf7132b70008fc78666a9e7d20ee7cfc
M: be34587acf7132b70008fc78666a9e7d20ee7cfc 127.0.0.1:7001
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 65e44433530c8acbd81a301958f939bb42921820 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 7d6334bebbca4386e775b7bbfdb29d758f537528
S: 73645156ab1df9fd4fdc9ee2ee0b6750238381ec 127.0.0.1:7005
   slots: (0 slots) slave
   replicates 09ed839f9c1c06f0b350426cf76a69793e9c97f3
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

```

## Retrieving cluster information
可以對 cluster 任一 node 執行指令
### cluster nodes
可以確認 cluster 現在的 node、每個 node 的 role 以及分配的 slot
```bash
$   ./redis-5.0.10/src/redis-cli -p 7000 cluster nodes
7d6334bebbca4386e775b7bbfdb29d758f537528 127.0.0.1:7002@17002 master - 0 1612880124787 3 connected 10923-16383
8a194deaa040bd97ac5fc425a13f394de91f5bcf 127.0.0.1:7003@17003 slave be34587acf7132b70008fc78666a9e7d20ee7cfc 0 1612880124588 4 connected
be34587acf7132b70008fc78666a9e7d20ee7cfc 127.0.0.1:7001@17001 master - 0 1612880124588 2 connected 5461-10922
65e44433530c8acbd81a301958f939bb42921820 127.0.0.1:7004@17004 slave 7d6334bebbca4386e775b7bbfdb29d758f537528 0 1612880123786 5 connected
73645156ab1df9fd4fdc9ee2ee0b6750238381ec 127.0.0.1:7005@17005 slave 09ed839f9c1c06f0b350426cf76a69793e9c97f3 0 1612880124287 6 connected
09ed839f9c1c06f0b350426cf76a69793e9c97f3 127.0.0.1:7000@17000 myself,master - 0 1612880123000 1 connected 0-5460
```
### cluster info
可以確認 cluster 的狀態

```bash
$   ./redis-5.0.10/src/redis-cli   -c -p 7000 cluster info

cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:608
cluster_stats_messages_pong_sent:622
cluster_stats_messages_sent:1230
cluster_stats_messages_ping_received:617
cluster_stats_messages_pong_received:608
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:1230

```
###   Stop Instance
starting up redis instances
```bash
./redis-5.0.10/src/redis-cli -p 7000 shutdown
./redis-5.0.10/src/redis-cli -p 7001 shutdown
./redis-5.0.10/src/redis-cli -p 7002 shutdown
./redis-5.0.10/src/redis-cli -p 7003 shutdown
./redis-5.0.10/src/redis-cli -p 7004 shutdown
./redis-5.0.10/src/redis-cli -p 7005 shutdown

```

#  Build Script
如果主要的測試是放在 client 端，純粹只是想要有個 redis cluster 提供服務，照上面方式逐一建立 config 、啟動 redis 再將所有 redis instance join 為 cluster 的動作就變得有些擾人，這時就可以使用內建的 shell 來快速建立 redis cluster

需要在 ***./utils/create-cluster/create-cluster.sh*** 所在目錄下執行 ***create-cluster.sh*** 才不會遇到路徑問題

啟動 redis instance (3 master + 3 slave)

1.預設會建立 30001,30002,30003,30004,30005,30006 六個 redis instance

```bash
$    cd redis-5.0.10/utils/create-cluster/
$   ./create-cluster start
Starting 30001
Starting 30002
Starting 30003
Starting 30004
Starting 30005
Starting 30006
```

2.建立 redis cluster
```bash
$ ./create-cluster create

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:30005 to 127.0.0.1:30001
Adding replica 127.0.0.1:30006 to 127.0.0.1:30002
Adding replica 127.0.0.1:30004 to 127.0.0.1:30003
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: b8fbb5e69211a291ae4e53a522bde5e5a620bc77 127.0.0.1:30001
   slots:[0-5460] (5461 slots) master
M: b27d43d89de3700563477504c7339ed27c84d123 127.0.0.1:30002
   slots:[5461-10922] (5462 slots) master
M: f51abc4aa25d6ec287fc33b148649068d09a6170 127.0.0.1:30003
   slots:[10923-16383] (5461 slots) master
S: d4d038a3f2cf44df113fea12bda437fdaf8000bc 127.0.0.1:30004
   replicates b8fbb5e69211a291ae4e53a522bde5e5a620bc77
S: 24a61bffda23ca680cca01b771266aa94ac15a88 127.0.0.1:30005
   replicates b27d43d89de3700563477504c7339ed27c84d123
S: 674b2f2cc822679ddfe3ed13bbb3f00fc04b97c4 127.0.0.1:30006
   replicates f51abc4aa25d6ec287fc33b148649068d09a6170
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join

>>> Performing Cluster Check (using node 127.0.0.1:30001)
M: b8fbb5e69211a291ae4e53a522bde5e5a620bc77 127.0.0.1:30001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 674b2f2cc822679ddfe3ed13bbb3f00fc04b97c4 127.0.0.1:30006
   slots: (0 slots) slave
   replicates f51abc4aa25d6ec287fc33b148649068d09a6170
S: d4d038a3f2cf44df113fea12bda437fdaf8000bc 127.0.0.1:30004
   slots: (0 slots) slave
   replicates b8fbb5e69211a291ae4e53a522bde5e5a620bc77
M: b27d43d89de3700563477504c7339ed27c84d123 127.0.0.1:30002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: f51abc4aa25d6ec287fc33b148649068d09a6170 127.0.0.1:30003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 24a61bffda23ca680cca01b771266aa94ac15a88 127.0.0.1:30005
   slots: (0 slots) slave
   replicates b27d43d89de3700563477504c7339ed27c84d123
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
3.停止 redis cluster
```bash
 $ ./create-cluster stop
Stopping 30001
Stopping 30002
Stopping 30003
Stopping 30004
Stopping 30005
Stopping 30006

```
4.完整清除 redis cluster 相關設定

```bash
./create-cluster clean
```