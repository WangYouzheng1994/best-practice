# Redis 7.2 Cluster 生产级配置部署手册

> **Author:** 王有政  
> **日期:** 2026-06-15  
> **适用系统:** CentOS 7.9 (x86_64)  
> **Redis 版本:** 7.2.7 (BSD-3 许可证)  
> **文档定位:** Redis Cluster 分片集群生产级部署。本文以 3 主 3 从为最小生产拓扑，适用于需要水平扩展容量和吞吐的 Redis 场景。

---

## 目录

1. [适用范围与架构原则](#1-适用范围与架构原则)
2. [推荐拓扑](#2-推荐拓扑)
3. [部署前置要求](#3-部署前置要求)
4. [节点基础配置](#4-节点基础配置)
5. [Cluster Redis 配置](#5-cluster-redis-配置)
6. [ACL 安全配置](#6-acl-安全配置)
7. [systemd 服务配置](#7-systemd-服务配置)
8. [防火墙与端口规划](#8-防火墙与端口规划)
9. [启动节点与创建集群](#9-启动节点与创建集群)
10. [集群验证](#10-集群验证)
11. [故障切换演练](#11-故障切换演练)
12. [扩容、缩容与迁移原则](#12-扩容缩容与迁移原则)
13. [生产验证清单](#13-生产验证清单)
14. [运维监控指标](#14-运维监控指标)
15. [变更与回滚建议](#15-变更与回滚建议)

---

## 1. 适用范围与架构原则

Redis Cluster 适用于以下场景：

- 单机 Redis 内存容量不足；
- 单机 Redis CPU 或网络吞吐不足；
- 需要按 hash slot 水平分片；
- 业务客户端支持 Redis Cluster 协议；
- 可接受 Redis Cluster 的多 key 操作限制。

Redis Cluster 生产原则：

1. **生产最小拓扑为 3 主 3 从。** 不建议生产使用 3 主 0 从。
2. **主从节点必须分散在不同故障域。** 同一主节点的 replica 不应与 master 位于同一台机器。
3. **客户端必须支持 Redis Cluster。** 普通单机客户端无法正确处理 `MOVED`、`ASK` 重定向。
4. **跨 slot 多 key 命令受限制。** 多 key 操作必须使用 hash tag 保证 key 在同一 slot。
5. **禁止公网访问 Redis 服务端口和 Cluster Bus 端口。**
6. **所有节点配置应保持一致。** 除 `bind`、`cluster-announce-ip`、日志路径等节点身份信息外，核心配置应统一。
7. **必须预留扩容容量。** Redis Cluster 扩容涉及 slot 迁移，生产应低峰操作。
8. **Redis Cluster 不等于强一致存储。** 主从复制仍是异步复制，故障时可能丢失少量已确认写入。

不适合 Redis Cluster 的场景：

- 客户端或中间件不支持 Cluster；
- 大量跨 key 事务、Lua、多 key 原子操作；
- key 分布极度不均且无法改造；
- 希望用 Redis 承载强一致核心数据。

---

## 2. 推荐拓扑

推荐 6 节点最小生产拓扑：

| 节点 | 角色 | 主机名 | 内网 IP | Redis 端口 | Cluster Bus 端口 |
|---|---|---|---|---:|---:|
| node-1 | Master | redis-cluster-01 | 10.10.30.11 | 6379 | 16379 |
| node-2 | Master | redis-cluster-02 | 10.10.30.12 | 6379 | 16379 |
| node-3 | Master | redis-cluster-03 | 10.10.30.13 | 6379 | 16379 |
| node-4 | Replica | redis-cluster-04 | 10.10.30.14 | 6379 | 16379 |
| node-5 | Replica | redis-cluster-05 | 10.10.30.15 | 6379 | 16379 |
| node-6 | Replica | redis-cluster-06 | 10.10.30.16 | 6379 | 16379 |

Redis Cluster 默认有 `16384` 个 hash slot，创建集群时会分配到 3 个 master：

| Master | Slot 范围示例 |
|---|---|
| node-1 | `0-5460` |
| node-2 | `5461-10922` |
| node-3 | `10923-16383` |

> 实际 slot 分配以 `redis-cli --cluster create` 输出为准。

---

## 3. 部署前置要求

每个节点必须先完成 `Redis7.2单体生产级配置部署手册.md` 中的基础规范：

- Redis 7.2 编译安装；
- 关闭 THP；
- 配置 `vm.overcommit_memory=1`；
- 配置 `net.core.somaxconn`；
- 配置 Swap 策略；
- 创建 `redis` 用户；
- 创建数据、日志、运行目录；
- systemd 托管；
- 禁止公网访问。

检查节点互通：

```bash
for ip in 10.10.30.11 10.10.30.12 10.10.30.13 10.10.30.14 10.10.30.15 10.10.30.16; do
  ping -c 3 $ip
done
```

检查端口互通，Redis Cluster 节点间必须同时开放 Redis 服务端口和 Cluster Bus 端口：

```bash
for ip in 10.10.30.11 10.10.30.12 10.10.30.13 10.10.30.14 10.10.30.15 10.10.30.16; do
  nc -vz $ip 6379
  nc -vz $ip 16379
done
```

---

## 4. 节点基础配置

每个节点执行：

```bash
id redis || useradd -r -s /sbin/nologin redis
mkdir -p /etc/redis
mkdir -p /data/redis/data
mkdir -p /data/redis/logs
mkdir -p /data/redis/run
mkdir -p /data/redis/backup
chown -R redis:redis /data/redis
chown root:redis /etc/redis
chmod 750 /etc/redis /data/redis /data/redis/data /data/redis/logs /data/redis/run /data/redis/backup
```

准备配置文件：

```bash
cp redis.conf /etc/redis/redis.conf
touch /etc/redis/users.acl
chown root:redis /etc/redis/redis.conf /etc/redis/users.acl
chmod 640 /etc/redis/redis.conf /etc/redis/users.acl
```

> `nodes.conf` 不要手工创建在 `/etc/redis` 下。Redis Cluster 会自动维护该文件，建议放在 Redis 数据目录中，由 `redis` 用户读写。

---

## 5. Cluster Redis 配置

以下配置为模板。每个节点必须替换：

- `bind` 中的本机内网 IP；
- `cluster-announce-ip`；
- 如多实例部署，还需替换端口、目录、日志文件和 `cluster-config-file`。

### 5.1 node-1 示例配置

`redis-cluster-01` 的 `/etc/redis/redis.conf`：

```conf
# ==== 基础运行 ====
daemonize no
supervised systemd
pidfile /data/redis/run/redis.pid
loglevel notice
logfile /data/redis/logs/redis.log
dir /data/redis/data
databases 1
always-show-logo no

# ==== 网络 ====
bind 127.0.0.1 10.10.30.11
protected-mode yes
port 6379
tcp-backlog 511
timeout 300
tcp-keepalive 300

# ==== ACL ====
aclfile /etc/redis/users.acl

# ==== Cluster ====
cluster-enabled yes
cluster-config-file /data/redis/data/nodes-6379.conf
cluster-node-timeout 15000
cluster-port 16379
cluster-announce-ip 10.10.30.11
cluster-announce-port 6379
cluster-announce-bus-port 16379
cluster-require-full-coverage yes
cluster-allow-reads-when-down no
cluster-migration-barrier 1

# ==== 复制认证 ====
masteruser replica
masterauth ReplaceWith_Replica_StrongPassword_2026

# ==== 内存 ====
maxmemory 8gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10

# ==== 持久化 ====
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 512mb
aof-use-rdb-preamble yes

# ==== 慢日志与延迟 ====
slowlog-log-slower-than 10000
slowlog-max-len 1024
latency-monitor-threshold 100

# ==== 客户端 ====
maxclients 50000
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# ==== 性能 ====
io-threads 4
io-threads-do-reads no
hz 10
dynamic-hz yes
activerehashing yes
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes
```

### 5.2 其他节点替换项

| 节点 | `bind` 内网 IP | `cluster-announce-ip` |
|---|---|---|
| node-1 | `10.10.30.11` | `10.10.30.11` |
| node-2 | `10.10.30.12` | `10.10.30.12` |
| node-3 | `10.10.30.13` | `10.10.30.13` |
| node-4 | `10.10.30.14` | `10.10.30.14` |
| node-5 | `10.10.30.15` | `10.10.30.15` |
| node-6 | `10.10.30.16` | `10.10.30.16` |

### 5.3 Cluster 参数说明

| 参数 | 建议值 | 说明 |
|---|---:|---|
| `cluster-enabled` | `yes` | 启用 Redis Cluster |
| `cluster-config-file` | 数据目录下 | Redis 自动维护，不能多人手工编辑 |
| `cluster-node-timeout` | `15000` | 节点失联判定基础超时时间，单位毫秒 |
| `cluster-port` | `16379` | Cluster Bus 端口 |
| `cluster-announce-ip` | 本机内网 IP | 节点对外声明地址，NAT/多网卡环境必须显式设置 |
| `cluster-require-full-coverage` | `yes` | 任意 slot 不可用时集群停止服务，避免部分数据不可用但业务误以为成功 |
| `cluster-allow-reads-when-down` | `no` | 集群不可用时禁止继续读，避免读到不一致数据 |
| `cluster-migration-barrier` | `1` | 至少保留 1 个 replica 后才允许迁移 replica |

---

## 6. ACL 安全配置

六个节点的 `/etc/redis/users.acl` 保持一致：

```acl
user default off
user app on >ReplaceWith_App_StrongPassword_2026 ~* +@read +@write +@connection +@transaction +@scripting -@admin -@dangerous -keys -flushdb -flushall
user monitor on >ReplaceWith_Monitor_StrongPassword_2026 ~* +ping +info +slowlog +client|getname +client|id +client|list +config|get +cluster|info +cluster|nodes +cluster|slots
user replica on >ReplaceWith_Replica_StrongPassword_2026 ~* +psync +replconf +ping
user clusteradmin on >ReplaceWith_ClusterAdmin_StrongPassword_2026 ~* +@all
```

说明：

- `app` 用于业务客户端；
- `monitor` 用于监控巡检；
- `replica` 用于主从复制认证；
- `clusteradmin` 仅用于创建集群、扩缩容、reshard、故障处理，不应给业务使用；
- 所有节点必须使用一致的 ACL 用户和密码，否则故障转移后复制或客户端访问可能失败。

redis-cli 操作时不建议使用 `-a` 明文参数，推荐：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER INFO
unset REDISCLI_AUTH
```

---

## 7. systemd 服务配置

六个节点创建 `/etc/systemd/system/redis.service`：

```bash
cat > /etc/systemd/system/redis.service << 'EOF'
[Unit]
Description=Redis 7.2 Cluster Node
Documentation=https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
After=network-online.target disable-thp.service
Wants=network-online.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/data/module/redis7.2/bin/redis-server /etc/redis/redis.conf --supervised systemd --daemonize no
ExecStop=/bin/kill -s TERM $MAINPID
TimeoutStartSec=30
TimeoutStopSec=60
Restart=always
RestartSec=3
LimitNOFILE=100032
PrivateTmp=true
ProtectHome=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable redis
```

---

## 8. 防火墙与端口规划

Redis Cluster 必须开放两个端口：

| 端口 | 用途 | 访问来源 |
|---:|---|---|
| 6379 | Redis 客户端端口 | 应用服务器、Redis 集群节点、监控节点、运维跳板机 |
| 16379 | Cluster Bus 端口 | 仅 Redis 集群节点之间 |

> 如果 Redis 端口不是 6379，Cluster Bus 默认通常是 Redis 端口 + 10000。本文显式配置 `cluster-port 16379`，避免歧义。

firewalld 示例：

```bash
firewall-cmd --permanent --new-zone=redis-cluster
firewall-cmd --permanent --zone=redis-cluster --add-source=10.10.30.0/24
firewall-cmd --permanent --zone=redis-cluster --add-source=10.10.20.0/24
firewall-cmd --permanent --zone=redis-cluster --add-port=6379/tcp
firewall-cmd --permanent --zone=redis-cluster --add-port=16379/tcp
firewall-cmd --reload
firewall-cmd --zone=redis-cluster --list-all
```

安全要求：

```text
禁止 0.0.0.0/0 -> 6379
禁止 0.0.0.0/0 -> 16379
应用服务器不需要访问 16379
```

---

## 9. 启动节点与创建集群

### 9.1 启动所有 Redis 节点

六个节点分别执行：

```bash
systemctl start redis
systemctl status redis --no-pager
```

确认每个节点都是 cluster 模式：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER INFO
unset REDISCLI_AUTH
```

创建集群前，`cluster_state` 可能不是 `ok`，这是正常的。

### 9.2 创建 3 主 3 从集群

在任意一台能访问六个节点的机器上执行：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli --cluster create \
  10.10.30.11:6379 \
  10.10.30.12:6379 \
  10.10.30.13:6379 \
  10.10.30.14:6379 \
  10.10.30.15:6379 \
  10.10.30.16:6379 \
  --cluster-replicas 1 \
  --cluster-yes \
  --user clusteradmin
unset REDISCLI_AUTH
```

说明：

- `--cluster-replicas 1` 表示每个 master 配 1 个 replica；
- `--cluster-yes` 表示自动确认，首次生产执行可先去掉该参数，人工核对分配计划；
- 如果 redis-cli 版本不支持 `--user`，应升级到与 Redis 7.2 匹配的 redis-cli。

### 9.3 客户端连接方式

业务客户端必须启用 Cluster 模式，并配置多个 startup nodes：

```text
10.10.30.11:6379,10.10.30.12:6379,10.10.30.13:6379,10.10.30.14:6379,10.10.30.15:6379,10.10.30.16:6379
username: app
password: ReplaceWith_App_StrongPassword_2026
cluster mode: enabled
```

redis-cli 访问集群必须使用 `-c`：

```bash
export REDISCLI_AUTH='ReplaceWith_App_StrongPassword_2026'
redis-cli -c -h 10.10.30.11 -p 6379 --user app SET cluster:check ok EX 300
redis-cli -c -h 10.10.30.11 -p 6379 --user app GET cluster:check
unset REDISCLI_AUTH
```

---

## 10. 集群验证

### 10.1 查看集群状态

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER INFO
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER NODES
unset REDISCLI_AUTH
```

期望：

```text
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_known_nodes:6
cluster_size:3
```

### 10.2 检查 slot 分布

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli --cluster check 10.10.30.11:6379 --user clusteradmin
unset REDISCLI_AUTH
```

期望：

```text
[OK] All nodes agree about slots configuration.
[OK] All 16384 slots covered.
```

### 10.3 验证 MOVED 重定向

不使用 `-c` 时，访问非本节点 slot 可能返回：

```text
MOVED <slot> <ip>:<port>
```

这是 Redis Cluster 正常行为。业务客户端必须支持自动处理 `MOVED` 和 `ASK`。

### 10.4 验证 hash tag

跨 slot 多 key 命令会失败：

```bash
MSET user:1:name a user:2:name b
```

可能返回：

```text
CROSSSLOT Keys in request don't hash to the same slot
```

需要使用 hash tag：

```bash
MSET user:{1001}:name a user:{1001}:age 20
```

---

## 11. 故障切换演练

### 11.1 停止一个 master

先确认 master 节点：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER NODES
unset REDISCLI_AUTH
```

在某个 master 节点上执行：

```bash
systemctl stop redis
```

观察集群状态：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.12 -p 6379 --user clusteradmin CLUSTER INFO
redis-cli -c -h 10.10.30.12 -p 6379 --user clusteradmin CLUSTER NODES
unset REDISCLI_AUTH
```

期望：

- 对应 replica 被提升为 master；
- `cluster_state` 恢复为 `ok`；
- slot 仍然全部覆盖。

### 11.2 恢复旧 master

在旧 master 节点执行：

```bash
systemctl start redis
```

恢复后旧 master 通常会作为新 master 的 replica 加入集群。确认：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.12 -p 6379 --user clusteradmin CLUSTER NODES
unset REDISCLI_AUTH
```

### 11.3 风险说明

Redis Cluster 故障切换不是强一致切换。以下情况可能产生数据丢失：

- master 写入成功后尚未复制到 replica 即宕机；
- 网络分区导致客户端仍短时间写入旧 master；
- replica 复制延迟较大时发生提升。

如业务对写入丢失极敏感，Redis Cluster 不应作为唯一事实源。

---

## 12. 扩容、缩容与迁移原则

### 12.1 扩容原则

扩容一般流程：

1. 新增 Redis 节点并完成基础配置；
2. 启动新节点；
3. 使用 `redis-cli --cluster add-node` 加入集群；
4. 使用 `redis-cli --cluster reshard` 迁移 slot；
5. 为新增 master 添加 replica；
6. 验证 slot 覆盖和主从关系。

示例：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli --cluster add-node 10.10.30.17:6379 10.10.30.11:6379 --user clusteradmin
redis-cli --cluster reshard 10.10.30.11:6379 --user clusteradmin
unset REDISCLI_AUTH
```

### 12.2 缩容原则

缩容必须先迁移被删除 master 上的 slot，再删除节点：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli --cluster reshard 10.10.30.11:6379 --user clusteradmin
redis-cli --cluster del-node 10.10.30.11:6379 <node-id> --user clusteradmin
unset REDISCLI_AUTH
```

注意：

- 不要直接停止仍持有 slot 的 master；
- 缩容前必须确认 slot 已迁移完成；
- 缩容操作应在业务低峰执行；
- 操作前必须备份配置和数据。

### 12.3 大 key 与热 key 风险

Cluster 只能按 key 分片，不能拆分单个大 key。生产应避免：

- 单个 hash/list/set/zset 过大；
- 所有热点 key 因 hash tag 落在同一 slot；
- 过度使用同一个 hash tag。

建议：

```text
单 key value 控制在业务可接受范围内，避免 MB 级大 key。
集合类 key 控制元素数量，必要时按业务维度拆分。
热点 key 应使用本地缓存、请求合并或业务拆分降低压力。
```

---

## 13. 生产验证清单

| 检查项 | 期望 |
|---|---|
| 节点数量 | 至少 6 个 |
| Master 数量 | 至少 3 个 |
| 每个 Master | 至少 1 个 Replica |
| `cluster_state` | `ok` |
| slot 覆盖 | `16384` 全覆盖 |
| Redis 服务端口 | 仅内网访问 |
| Cluster Bus 端口 | 仅集群节点访问 |
| `cluster-announce-ip` | 正确内网 IP |
| `cluster-config-file` | 位于数据目录且 Redis 可写 |
| ACL | 所有节点一致 |
| `default` 用户 | `off` |
| 应用客户端 | Cluster 模式 |
| 跨 slot 操作 | 已完成业务改造 |
| 持久化 | 符合数据可靠性要求 |
| 故障切换演练 | 已完成 |
| 扩缩容演练 | 预生产已验证 |

---

## 14. 运维监控指标

### 14.1 Cluster 状态

重点命令：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER INFO
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER NODES
redis-cli --cluster check 10.10.30.11:6379 --user clusteradmin
unset REDISCLI_AUTH
```

重点指标：

| 指标 | 告警条件 |
|---|---|
| `cluster_state` | 不为 `ok` |
| `cluster_slots_fail` | 大于 `0` |
| `cluster_slots_pfail` | 大于 `0` |
| `cluster_known_nodes` | 小于预期 |
| `cluster_size` | 小于预期 master 数 |
| slot 覆盖 | 小于 `16384` |

### 14.2 节点状态

每个节点需要监控：

| 指标 | 风险说明 |
|---|---|
| `used_memory` | 内存使用量 |
| `used_memory_rss` | 进程实际内存 |
| `mem_fragmentation_ratio` | 内存碎片率 |
| `connected_clients` | 客户端连接数 |
| `blocked_clients` | 阻塞客户端 |
| `evicted_keys` | key 淘汰数量 |
| `rejected_connections` | 连接被拒绝 |
| `instantaneous_ops_per_sec` | QPS |
| `latest_fork_usec` | fork 延迟 |
| `master_link_status` | replica 到 master 的复制状态 |
| `master_last_io_seconds_ago` | 复制链路空闲时间 |

### 14.3 业务侧指标

必须监控客户端侧：

- `MOVED` 重定向数量；
- `ASK` 重定向数量；
- 连接池耗尽；
- 命令超时；
- `CROSSSLOT` 错误；
- 热 key；
- 大 key；
- 单 slot 流量倾斜。

---

## 15. 变更与回滚建议

### 15.1 变更前备份

每个节点执行：

```bash
cp /etc/redis/redis.conf /etc/redis/redis.conf.bak.$(date +%F-%H%M%S)
cp /etc/redis/users.acl /etc/redis/users.acl.bak.$(date +%F-%H%M%S)
cp /data/redis/data/nodes-6379.conf /data/redis/data/nodes-6379.conf.bak.$(date +%F-%H%M%S)
```

### 15.2 禁止事项

生产禁止：

1. 手工编辑运行中的 `nodes-6379.conf`；
2. 同时重启多数 master；
3. 直接停止持有 slot 的 master 做缩容；
4. 在未验证客户端支持 Cluster 的情况下迁移业务；
5. 对公网开放 `6379` 或 `16379`；
6. 在高峰期执行大规模 reshard；
7. 使用不一致的 ACL 文件启动部分节点。

### 15.3 滚动重启建议

单节点滚动重启：

```bash
systemctl restart redis
systemctl status redis --no-pager
```

每重启一个节点后必须确认：

```bash
export REDISCLI_AUTH='ReplaceWith_ClusterAdmin_StrongPassword_2026'
redis-cli -c -h 10.10.30.11 -p 6379 --user clusteradmin CLUSTER INFO
redis-cli --cluster check 10.10.30.11:6379 --user clusteradmin
unset REDISCLI_AUTH
```

确认 `cluster_state:ok` 后再处理下一个节点。

### 15.4 回滚原则

- 配置变更失败：回滚 `redis.conf` 和 `users.acl` 后重启单节点；
- 创建集群失败：确认所有节点无业务数据后，可停止 Redis、清理 `nodes-6379.conf`、清空测试数据，再重新创建；
- 扩容失败：优先检查新节点状态和 slot 分布，不要直接删除未知状态节点；
- 缩容失败：先停止继续迁移，执行 `redis-cli --cluster check` 明确 slot 状态后再处理。

> 注意：清空数据目录是破坏性操作，只能在确认节点无生产数据或已有备份且获准后执行。
