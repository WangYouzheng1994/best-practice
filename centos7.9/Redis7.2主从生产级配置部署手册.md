# Redis 7.2 主从生产级配置部署手册

> **Author:** 王有政  
> **日期:** 2026-07-15<br>
> **适用系统:** CentOS 7.9 (x86_64)  
> **Redis 版本:** 7.2.7 (BSD-3 许可证)  
> **文档定位:** Redis 主从复制生产级部署。本文以“一主两从 + 三 Sentinel”为推荐生产形态；如只部署主从而不部署 Sentinel，则只能提供读扩展和数据冗余，不能提供自动故障转移。

---

## 目录

1. [适用范围与架构原则](#1-适用范围与架构原则)
2. [推荐拓扑](#2-推荐拓扑)
3. [部署前置要求](#3-部署前置要求)
4. [节点基础配置](#4-节点基础配置)
5. [主节点 Redis 配置](#5-主节点-redis-配置)
6. [从节点 Redis 配置](#6-从节点-redis-配置)
7. [Sentinel 哨兵配置](#7-sentinel-哨兵配置)
8. [systemd 服务配置](#8-systemd-服务配置)
9. [网络边界与端口](#9-网络边界与端口)
10. [启动顺序与验证](#10-启动顺序与验证)
11. [变更与回滚建议](#11-变更与回滚建议)

## 1. 适用范围与架构原则

本文适用于 Redis 7.2 主从复制生产部署，目标是实现：

- 主节点承载写请求；
- 从节点承载只读请求或作为故障切换候选；
- Sentinel 监控主从状态并在主节点故障时自动选主；
- 应用侧通过 Sentinel 或具备 Sentinel 支持的客户端发现当前主节点。

重要原则：

1. **主从复制不等于高可用。** 只有主从而没有 Sentinel 或外部故障转移系统时，主节点故障后需要人工切换。
2. **Sentinel 至少 3 个节点。** 生产环境不要部署 1 个或 2 个 Sentinel。
3. **Sentinel 节点应分布在不同故障域。** 不要全部部署在同一台机器。
4. **应用必须支持 Sentinel。** 否则 Sentinel 完成故障切换后，应用可能仍连接旧主节点。
5. **写入一致性不是强一致。** Redis 主从复制默认异步复制，主节点故障时可能丢失少量已确认写入。
6. **不要在同一台机器上同时运行主节点和全部从节点。** 这不能抵抗主机故障。
7. **Redis 进程不得使用 root 用户运行。**
8. **禁止公网访问 Redis 与 Sentinel 端口。**
9. **禁止使用 Redis 默认端口 `6379`。** Redis 服务端口必须遵循《服务器端口统一规划规范.md》，默认使用 `16300 ~ 16399` 端口段，主从标准端口为 `16379`；Sentinel 使用 `26300 ~ 26399` 端口段，标准端口为 `26379`。

---

## 2. 推荐拓扑

推荐生产最小拓扑：

| 角色 | 主机名 | 内网 IP | Redis 端口 | Sentinel 端口 |
|---|---|---|---:|---:|
| Master | redis-01 | 10.10.10.11 | 16379 | 26379 |
| Replica | redis-02 | 10.10.10.12 | 16379 | 26379 |
| Replica | redis-03 | 10.10.10.13 | 16379 | 26379 |

Sentinel 监控名称：

```text
mymaster
```

故障判定票数：

```text
quorum = 2
```

推荐读写策略：

| 流量类型 | 推荐连接目标 |
|---|---|
| 写请求 | 当前 Master |
| 强一致读 | 当前 Master |
| 可接受复制延迟的读 | Replica |
| 管理与故障发现 | Sentinel |

> 注意：如果业务不能接受复制延迟，不应把读请求分散到 Replica。

---

## 3. 部署前置要求

每个节点都必须先完成 `Redis7.2单体生产级配置部署手册.md` 中的基础配置，包括：

- Redis 7.2 编译安装；
- 关闭 THP；
- 配置 `vm.overcommit_memory=1`；
- 配置 `net.core.somaxconn`；
- 配置 Swap 策略；
- 创建 `redis` 用户；
- 创建 `/etc/redis`、`/data/redis/data`、`/data/redis/logs`、`/data/redis/run`；
- 配置文件权限为 `root:redis 640`；
- systemd 托管；
- 禁止公网访问。

检查三台节点互通：

```bash
ping -c 3 10.10.10.11
ping -c 3 10.10.10.12
ping -c 3 10.10.10.13
```

检查端口连通性，需在节点间互测：

```bash
nc -vz 10.10.10.11 16379
nc -vz 10.10.10.12 16379
nc -vz 10.10.10.13 16379
nc -vz 10.10.10.11 26379
nc -vz 10.10.10.12 26379
nc -vz 10.10.10.13 26379
```

---

## 4. 节点基础配置

### 4.1 目录规划

每个节点执行：

```bash
mkdir -p /etc/redis
mkdir -p /data/redis/data
mkdir -p /data/redis/logs
mkdir -p /data/redis/run
mkdir -p /data/redis/sentinel
mkdir -p /data/redis/backup
chown -R redis:redis /data/redis
chown root:redis /etc/redis
chmod 750 /etc/redis /data/redis /data/redis/data /data/redis/logs /data/redis/run /data/redis/sentinel /data/redis/backup
```

### 4.2 配置文件

每个节点准备：

```text
/etc/redis/redis.conf
/etc/redis/users.acl
/etc/redis/sentinel.conf
```

权限：

```bash
chown root:redis /etc/redis/redis.conf /etc/redis/users.acl /etc/redis/sentinel.conf
chmod 640 /etc/redis/redis.conf /etc/redis/users.acl /etc/redis/sentinel.conf
```

Sentinel 会自动重写 `sentinel.conf`，因此生产中也可将 `sentinel.conf` 设置为 `redis:redis 640`。如果使用 `root:redis`，需要确认 Sentinel 进程具备写权限，否则故障切换状态无法持久化。推荐：

```bash
chown redis:redis /etc/redis/sentinel.conf
chmod 640 /etc/redis/sentinel.conf
```

---

## 5. 主节点 Redis 配置

在 `redis-01` 的 `/etc/redis/redis.conf` 中配置：

```conf
# ==== 基础运行 ====
daemonize no
supervised systemd
pidfile /data/redis/run/redis.pid
loglevel notice
logfile /data/redis/logs/redis.log
dir /data/redis/data
databases 16
always-show-logo no

# ==== 网络 ====
bind 127.0.0.1 10.10.10.11
protected-mode yes
port 16379
tcp-backlog 511
timeout 300
tcp-keepalive 300

# ==== ACL ====
aclfile /etc/redis/users.acl

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

# ==== 主从复制保护 ====
repl-backlog-size 256mb
repl-backlog-ttl 3600
min-replicas-to-write 1
min-replicas-max-lag 10

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

关键说明：

- `min-replicas-to-write 1` 表示至少 1 个从节点复制延迟不超过 `10` 秒时，主节点才继续接受写入；
- 该配置会提高安全性，但当所有从节点异常时，主节点会拒绝写入；
- 如果业务优先保证可写性，可改为不启用该配置，但需接受主节点故障时更大的数据丢失风险。

---

## 6. 从节点 Redis 配置

在 `redis-02` 的 `/etc/redis/redis.conf` 中配置：

```conf
# ==== 基础运行 ====
daemonize no
supervised systemd
pidfile /data/redis/run/redis.pid
loglevel notice
logfile /data/redis/logs/redis.log
dir /data/redis/data
databases 16
always-show-logo no

# ==== 网络 ====
bind 127.0.0.1 10.10.10.12
protected-mode yes
port 16379
tcp-backlog 511
timeout 300
tcp-keepalive 300

# ==== ACL ====
aclfile /etc/redis/users.acl

# ==== 从节点复制 ====
replicaof 10.10.10.11 16379
masteruser replica
masterauth ReplaceWith_Replica_StrongPassword_2026
replica-read-only yes
replica-serve-stale-data yes
repl-backlog-size 256mb
repl-backlog-ttl 3600

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
```

在 `redis-03` 上仅替换绑定 IP：

```conf
bind 127.0.0.1 10.10.10.13
replicaof 10.10.10.11 16379
```

### 6.1 从节点是否开启 AOF

主节点和从节点统一开启 AOF，保证故障切换后的数据恢复能力。

---

## 7. Sentinel 哨兵配置

### 7.1 Redis ACL 文件

三台 Redis 节点 `/etc/redis/users.acl` 保持一致：

```acl
user default off
user app on >ReplaceWith_App_StrongPassword_2026 ~* +@read +@write +@connection +@transaction +@scripting +info -@admin -@dangerous -keys -flushdb -flushall
user monitor on >ReplaceWith_Monitor_StrongPassword_2026 ~* +ping +info +slowlog +client|getname +client|id +client|list +config|get
user replica on >ReplaceWith_Replica_StrongPassword_2026 ~* +psync +replconf +ping
user sentinel on >ReplaceWith_Sentinel_StrongPassword_2026 ~* +@all
```

说明：

- `replica` 用户用于从节点向主节点复制；
- `sentinel` 用户用于 Sentinel 监控和故障切换；
- Sentinel 在故障切换期间需要执行配置变更相关命令，因此权限不能过窄；
- 密码必须在三台 Redis 节点上保持一致。

### 7.2 Sentinel 配置文件

三台节点分别创建 `/etc/redis/sentinel.conf`。

`redis-01`：

```conf
port 26379
bind 127.0.0.1 10.10.10.11
protected-mode yes
daemonize no
supervised systemd
pidfile /data/redis/run/redis-sentinel.pid
logfile /data/redis/logs/redis-sentinel.log
dir /data/redis/sentinel

sentinel monitor mymaster 10.10.10.11 16379 2
sentinel auth-user mymaster sentinel
sentinel auth-pass mymaster ReplaceWith_Sentinel_StrongPassword_2026
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 60000
sentinel deny-scripts-reconfig yes
```

`redis-02`：

```conf
port 26379
bind 127.0.0.1 10.10.10.12
protected-mode yes
daemonize no
supervised systemd
pidfile /data/redis/run/redis-sentinel.pid
logfile /data/redis/logs/redis-sentinel.log
dir /data/redis/sentinel

sentinel monitor mymaster 10.10.10.11 16379 2
sentinel auth-user mymaster sentinel
sentinel auth-pass mymaster ReplaceWith_Sentinel_StrongPassword_2026
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 60000
sentinel deny-scripts-reconfig yes
```

`redis-03`：

```conf
port 26379
bind 127.0.0.1 10.10.10.13
protected-mode yes
daemonize no
supervised systemd
pidfile /data/redis/run/redis-sentinel.pid
logfile /data/redis/logs/redis-sentinel.log
dir /data/redis/sentinel

sentinel monitor mymaster 10.10.10.11 16379 2
sentinel auth-user mymaster sentinel
sentinel auth-pass mymaster ReplaceWith_Sentinel_StrongPassword_2026
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 60000
sentinel deny-scripts-reconfig yes
```

> Sentinel 启动后会自动发现从节点，并会重写 `sentinel.conf`。这是正常行为，不要用配置管理工具强行覆盖运行中的 Sentinel 配置文件。

### 7.3 Sentinel 认证说明

Sentinel 访问 Redis 使用：

```conf
sentinel auth-user mymaster sentinel
sentinel auth-pass mymaster ReplaceWith_Sentinel_StrongPassword_2026
```

从节点复制主节点使用：

```conf
masteruser replica
masterauth ReplaceWith_Replica_StrongPassword_2026
```

应用访问 Redis 使用：

```text
app / ReplaceWith_App_StrongPassword_2026
```

不要混用应用用户、复制用户和 Sentinel 用户。

---

## 8. systemd 服务配置

### 8.1 Redis 服务

三台节点创建 `/etc/systemd/system/redis.service`：

```bash
cat > /etc/systemd/system/redis.service << 'EOF'
[Unit]
Description=Redis 7.2 Server
Documentation=https://redis.io/docs/
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
```

### 8.2 Sentinel 服务

三台节点创建 `/etc/systemd/system/redis-sentinel.service`：

```bash
cat > /etc/systemd/system/redis-sentinel.service << 'EOF'
[Unit]
Description=Redis 7.2 Sentinel
Documentation=https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
After=network-online.target redis.service
Wants=network-online.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/data/module/redis7.2/bin/redis-server /etc/redis/sentinel.conf --sentinel --supervised systemd --daemonize no
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
```

加载服务：

```bash
systemctl daemon-reload
systemctl enable redis redis-sentinel
```

---

## 9. 网络边界与端口

本文不维护本机 `firewalld` 规则，主机防火墙基线由《CentOS7.9生产环境初始化手册.md》统一规定。

Redis `16379` 仅允许应用服务器、Redis 节点、监控服务器访问；Sentinel `26379` 仅允许应用服务器、Sentinel 节点、运维跳板机访问。

部署前需要确认：

- 端口仅对可信内网开放，禁止从公网直接访问；
- 访问来源由外层防火墙、安全组、ACL、VPN、堡垒机或统一网络策略控制；
- 端口、来源、用途应登记到资产台账或 CMDB。

---

## 10. 启动顺序与验证

### 10.1 启动 Redis

先启动主节点：

```bash
systemctl start redis
systemctl status redis --no-pager
```

再启动两个从节点：

```bash
systemctl start redis
systemctl status redis --no-pager
```

### 10.4 启动 Sentinel

三台节点启动 Sentinel：

```bash
systemctl start redis-sentinel
systemctl status redis-sentinel --no-pager
```

验证 Sentinel：

```bash
redis-cli -h 10.10.10.11 -p 26379 SENTINEL masters
redis-cli -h 10.10.10.11 -p 26379 SENTINEL replicas mymaster
redis-cli -h 10.10.10.11 -p 26379 SENTINEL sentinels mymaster
```

期望：

- `num-other-sentinels` 至少为 `2`；
- 能看到两个 replicas；
- `flags` 不包含 `s_down`、`o_down`。

---

## 11. 变更与回滚建议

### 11.1 变更前备份

```bash
cp /etc/redis/redis.conf /etc/redis/redis.conf.bak.$(date +%F-%H%M%S)
cp /etc/redis/users.acl /etc/redis/users.acl.bak.$(date +%F-%H%M%S)
cp /etc/redis/sentinel.conf /etc/redis/sentinel.conf.bak.$(date +%F-%H%M%S)
```

### 11.2 配置变更顺序

推荐顺序：

1. 先改从节点；
2. 验证复制正常；
3. 再改主节点；
4. 最后改 Sentinel；
5. 执行故障切换演练。

### 11.3 回滚原则

- 如果从节点变更失败，优先回滚从节点配置；
- 如果主节点变更失败，先确认是否已经发生 Sentinel failover；
- 如果 Sentinel 配置异常，不要同时停止所有 Sentinel；
- 不要手工编辑运行中 Sentinel 自动重写后的主从状态，除非明确理解当前拓扑。

### 11.4 应用连接建议

应用应配置 Sentinel 地址列表，而不是固定连接某个 Redis 节点：

```text
sentinel nodes: 10.10.10.11:26379,10.10.10.12:26379,10.10.10.13:26379
master name: mymaster
username: app
password: ReplaceWith_App_StrongPassword_2026
```

如果应用不支持 Sentinel，只能通过人工或外部 VIP/LB 方案切换主节点。该模式不推荐作为生产高可用方案。
