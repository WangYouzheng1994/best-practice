# Redis 7.2 单体生产级配置部署手册

> **Author:** 王有政  
> **日期:** 2026-06-15  
> **适用系统:** CentOS 7.9 (x86_64)  
> **Redis 版本:** 7.2.7 (BSD-3 许可证)  
> **文档定位:** 单体 Redis 生产级配置。主从复制与 Redis Cluster 请拆分为独立文档维护。

---

## 目录

1. [适用范围与部署原则](#1-适用范围与部署原则)
2. [部署前检查](#2-部署前检查)
3. [OS 内核与资源限制配置](#3-os-内核与资源限制配置)
4. [目录、用户与权限](#4-目录用户与权限)
5. [Redis 配置文件准备](#5-redis-配置文件准备)
6. [单体 Redis 核心配置](#6-单体-redis-核心配置)
7. [ACL 安全配置](#7-acl-安全配置)
8. [systemd 服务配置](#8-systemd-服务配置)
9. [防火墙与网络访问控制](#9-防火墙与网络访问控制)
10. [启动与基础验证](#10-启动与基础验证)
11. [生产验证清单](#11-生产验证清单)
12. [运维检查命令](#12-运维检查命令)
13. [变更与回滚建议](#13-变更与回滚建议)

---

## 1. 适用范围与部署原则

本文用于部署 **单体 Redis 7.2**，适用于以下场景：

- 单机缓存服务；
- 开发、测试、预生产环境；
- 生产环境中可接受单点风险的非核心缓存；
- 后续作为主从、Sentinel、Cluster 部署的基础节点模板。

本文不覆盖以下内容：

- 主从复制；
- Sentinel 哨兵高可用；
- Redis Cluster 分片集群；
- 跨机房容灾。

生产级单体 Redis 必须遵循以下原则：

1. Redis 进程不得使用 `root` 用户运行；
2. 不得监听公网地址；
3. 不得默认使用 `bind 0.0.0.0`；
4. 必须启用认证与 ACL；
5. 必须设置 `maxmemory`，避免系统 OOM；
6. 必须关闭 Transparent Huge Pages；
7. 必须配置持久化策略，或者明确声明该实例是可丢失缓存；
8. 必须由 systemd 托管并设置开机自启；
9. 必须限制访问来源 IP；
10. 必须完成启动验证、持久化验证、安全验证和资源限制验证；
11. 禁止使用 Redis 默认端口 `6379`，必须遵循《服务器端口统一规划规范.md》，默认使用 `16300 ~ 16399` 端口段，单体标准端口为 `16379`。

---

## 2. 部署前检查

### 2.1 Redis 安装确认

确认 Redis 已完成源码编译安装。

```bash
redis-server --version
redis-cli --version
```

示例输出：

```text
Redis server v=7.2.7 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=xxxx
redis-cli 7.2.7
```

如果 Redis 使用自定义路径安装，确认二进制路径：

```bash
which redis-server
which redis-cli
```

本文默认 Redis 安装路径如下：

```text
/data/module/redis7.2/bin/redis-server
/data/module/redis7.2/bin/redis-cli
```

如实际路径不同，后续命令需替换为实际路径。

### 2.2 服务器资源建议

| 资源 | 生产建议 |
|---|---|
| CPU | 4 核及以上 |
| 内存 | 8 GB 及以上 |
| 数据盘 | 独立数据盘，建议 SSD |
| 文件系统 | xfs 或 ext4 |
| 网络 | 仅内网访问 |
| 时间同步 | 必须启用 chrony 或 ntpd |

检查基础信息：

```bash
hostnamectl
lscpu
free -h
lsblk
df -h
ip addr
```

---

## 3. OS 内核与资源限制配置

### 3.1 关闭 Transparent Huge Pages

THP 会导致 Redis 在 `BGSAVE`、AOF rewrite、fork 子进程时出现延迟尖刺，生产环境必须关闭。

查看当前状态：

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

临时关闭：

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
[ -f /sys/kernel/mm/transparent_hugepage/defrag ] && echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

创建 systemd 服务实现开机自动关闭：

```bash
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages for Redis
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=redis.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled; if [ -f /sys/kernel/mm/transparent_hugepage/defrag ]; then echo never > /sys/kernel/mm/transparent_hugepage/defrag; fi'

[Install]
WantedBy=basic.target
EOF

systemctl daemon-reload
systemctl enable disable-thp --now
```

验证：

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

期望 `never` 被 `[]` 包裹。

### 3.2 配置 sysctl 参数

不要直接反复向 `/etc/sysctl.conf` 追加配置。生产建议使用独立配置文件，便于维护和回滚。

```bash
cat > /etc/sysctl.d/99-redis.conf << 'EOF'
vm.overcommit_memory = 1
vm.swappiness = 1
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
EOF

sysctl --system
```

参数说明：

| 参数 | 建议值 | 说明 |
|---|---:|---|
| `vm.overcommit_memory` | `1` | 避免 Redis fork 子进程时因内存分配策略导致失败 |
| `vm.swappiness` | `1` | 尽量避免 Redis 内存页被换出 |
| `net.core.somaxconn` | `65535` | 提高 TCP accept 队列上限 |
| `net.ipv4.tcp_max_syn_backlog` | `65535` | 提高 SYN 队列上限 |
| `net.ipv4.tcp_keepalive_time` | `300` | TCP keepalive 探测时间 |
| `net.ipv4.tcp_keepalive_intvl` | `30` | TCP keepalive 探测间隔 |
| `net.ipv4.tcp_keepalive_probes` | `5` | TCP keepalive 探测次数 |

验证：

```bash
sysctl vm.overcommit_memory
sysctl vm.swappiness
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
```

### 3.3 Swap 策略

Redis 对延迟敏感，生产环境应尽量避免使用 Swap。

#### 方案 A：完全关闭 Swap

适用场景：物理内存充足，且 Redis 已正确设置 `maxmemory`。

```bash
swapon --show
cp /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
swapoff -a
sed -i.bak '/[[:space:]]swap[[:space:]]/ s/^/#/' /etc/fstab
swapon --show
```

#### 方案 B：保留 Swap，但降低使用倾向

适用场景：服务器内存较小，或运维规范要求保留 Swap。

```bash
sysctl vm.swappiness=1
sysctl vm.swappiness
```

#### 方案 C：不建议方案

不建议在 Redis 生产环境中保留默认 `vm.swappiness=30` 或 `60`。这会提高 Redis 内存页被换出的概率，造成严重延迟抖动。

**推荐：** 如果服务器内存充足并且 `maxmemory` 设置合理，选择方案 A；否则选择方案 B。

### 3.4 文件句柄数限制

Redis 每个客户端连接至少占用一个文件描述符。生产环境必须提高文件句柄限制。

```bash
cat > /etc/security/limits.d/99-redis.conf << 'EOF'
redis soft nofile 100032
redis hard nofile 100032
EOF
```

systemd 服务中也需要设置 `LimitNOFILE=100032`，见 [8. systemd 服务配置](#8-systemd-服务配置)。

---

## 4. 目录、用户与权限

### 4.1 创建 redis 用户

```bash
id redis || useradd -r -s /sbin/nologin redis
```

### 4.2 创建目录

```bash
mkdir -p /etc/redis
mkdir -p /data/redis/data
mkdir -p /data/redis/logs
mkdir -p /data/redis/run
mkdir -p /data/redis/backup
```

目录用途：

| 目录 | 用途 |
|---|---|
| `/etc/redis` | Redis 配置文件与 ACL 文件 |
| `/data/redis/data` | RDB、AOF、临时 rewrite 文件 |
| `/data/redis/logs` | Redis 日志 |
| `/data/redis/run` | PID 文件 |
| `/data/redis/backup` | 备份文件暂存目录 |

### 4.3 设置权限

```bash
chown -R redis:redis /data/redis
chown root:redis /etc/redis
chmod 750 /etc/redis
chmod 750 /data/redis
chmod 750 /data/redis/data /data/redis/logs /data/redis/run /data/redis/backup
```

配置文件建议归属 `root:redis`，Redis 进程只读配置，不应拥有修改配置文件的权限。

---

## 5. Redis 配置文件准备

从源码目录复制默认配置文件：

```bash
cp redis.conf /etc/redis/redis.conf
chown root:redis /etc/redis/redis.conf
chmod 640 /etc/redis/redis.conf
```

创建 ACL 文件：

```bash
touch /etc/redis/users.acl
chown root:redis /etc/redis/users.acl
chmod 640 /etc/redis/users.acl
```

> 说明：`users.acl` 中包含账号密码与权限规则，必须严格限制权限。

---

## 6. 单体 Redis 核心配置

编辑 `/etc/redis/redis.conf`，按以下内容检查和调整。

### 6.1 基础运行配置

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
```

### 6.2 网络配置

生产环境禁止默认绑定 `0.0.0.0`。应只绑定本机回环地址和 Redis 服务内网 IP。

```conf
# ==== 网络 ====
# 将 10.10.10.10 替换为本机实际内网 IP
bind 127.0.0.1 10.10.10.10
protected-mode yes
port 16379
tcp-backlog 511
timeout 300
tcp-keepalive 300
```

如 Redis 仅供本机应用访问，使用：

```conf
bind 127.0.0.1
protected-mode yes
port 16379
```

不允许作为生产默认值：

```conf
bind 0.0.0.0
```

### 6.3 内存上限配置

Redis 必须设置 `maxmemory`，避免 Redis 挤占系统内存导致 OOM。

```conf
# ==== 内存 ====
# 示例：16 GB 服务器可设置 8gb ~ 10gb，需结合业务写入量和持久化策略评估
maxmemory 8gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10
```

`maxmemory` 建议：

| 服务器内存 | 推荐 `maxmemory` |
|---:|---:|
| 8 GB | 4 GB ~ 5 GB |
| 16 GB | 8 GB ~ 10 GB |
| 32 GB | 18 GB ~ 22 GB |
| 64 GB | 38 GB ~ 45 GB |

需要为以下内容预留内存：

- 操作系统；
- Redis fork 时的 Copy-On-Write；
- AOF rewrite；
- RDB 快照；
- 客户端输出缓冲区；
- jemalloc 内存碎片；
- 监控 Agent 和系统服务。

### 6.4 淘汰策略选择

| 策略 | 适用场景 | 风险 |
|---|---|---|
| `allkeys-lfu` | 通用缓存、热点数据保护 | 所有 key 都可能被淘汰 |
| `allkeys-lru` | 通用缓存 | 不如 LFU 保护热点数据 |
| `volatile-lfu` | 只有设置 TTL 的 key 允许淘汰 | 未设置 TTL 的 key 可能导致 OOM |
| `volatile-lru` | 兼容传统缓存策略 | 未设置 TTL 的 key 可能导致 OOM |
| `noeviction` | 不能丢数据的状态类场景 | 写入达到上限后报错 |

推荐三种生产方案：

#### 方案 A：通用缓存实例

```conf
maxmemory-policy allkeys-lfu
```

适用于大多数缓存场景。**推荐作为单体缓存 Redis 的默认策略。**

#### 方案 B：只有带 TTL 的 key 可淘汰

```conf
maxmemory-policy volatile-lfu
```

适用于严格要求只有设置过期时间的缓存 key 才可被淘汰的场景。应用必须保证可淘汰 key 全部设置 TTL。

#### 方案 C：禁止淘汰

```conf
maxmemory-policy noeviction
```

适用于分布式锁、状态存储、队列等不能被 Redis 主动淘汰的场景。应用必须正确处理写入失败。

### 6.5 持久化配置

是否启用持久化取决于 Redis 数据是否允许丢失。

#### 方案 A：纯缓存，可完全重建

```conf
save ""
appendonly no
```

优点：性能最好。  
缺点：Redis 重启后数据全部丢失。

#### 方案 B：通用生产推荐，RDB + AOF everysec

```conf
# ==== RDB ====
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb

# ==== AOF ====
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 512mb
aof-use-rdb-preamble yes
```

优点：可靠性和性能平衡。  
缺点：极端情况下可能丢失约 1 秒数据。

#### 方案 C：强持久化

```conf
appendonly yes
appendfsync always
```

优点：数据丢失窗口最小。  
缺点：写入性能下降明显，不推荐普通缓存场景使用。

**推荐：** 单体生产缓存默认选择方案 B；如果业务明确允许丢失并可自动预热，选择方案 A。

### 6.6 性能相关配置

```conf
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

说明：

- Redis 主要命令执行仍是单线程模型；
- `io-threads` 主要用于网络 I/O，不能简单等同于 CPU 核数；
- 低并发场景不需要盲目提高 `io-threads`；
- 一般 4 核机器设置 `io-threads 2` 或 `4` 即可，8 核及以上可结合压测评估。

### 6.7 慢日志与延迟监控

```conf
# ==== 慢日志与延迟 ====
slowlog-log-slower-than 10000
slowlog-max-len 1024
latency-monitor-threshold 100
```

说明：

- `slowlog-log-slower-than 10000` 表示记录执行超过 10 ms 的命令；
- 低延迟业务可调整为 `5000`；
- 生产环境不建议设置为负数禁用慢日志。

### 6.8 客户端连接管理

```conf
# ==== 客户端 ====
maxclients 50000
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

要求：

```text
LimitNOFILE > maxclients + Redis 内部文件描述符开销
```

本文设置：

```text
LimitNOFILE=100032
maxclients=50000
```

符合生产要求。

### 6.9 禁用危险命令的策略

Redis 6+ 推荐优先使用 ACL 控制危险权限，不建议默认使用 `rename-command` 作为主要安全手段。

原因：

- `rename-command` 可能影响监控、备份、自动化运维工具；
- 部分客户端或平台组件可能依赖 `CONFIG`、`INFO` 等命令；
- ACL 权限模型更清晰，审计和维护成本更低。

如企业安全规范强制要求禁用危险命令，可在充分验证后增加：

```conf
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""
```

不建议默认重命名 `CONFIG`、`SHUTDOWN`，除非已经确认所有运维工具兼容。

---

## 7. ACL 安全配置

### 7.1 ACL 设计原则

生产环境建议：

1. 禁用 `default` 用户；
2. 为应用创建独立用户；
3. 为监控创建只读用户；
4. 不在命令行中直接暴露密码；
5. 不使用 `redis-cli -a 明文密码` 作为生产执行规范；
6. 密码必须由密码管理系统生成和保存。

### 7.2 创建 ACL 文件

编辑 `/etc/redis/users.acl`：

```acl
user default off
user app on >ReplaceWith_App_StrongPassword_2026 ~* +@read +@write +@connection +@transaction +@scripting -@admin -@dangerous -keys -flushdb -flushall
user monitor on >ReplaceWith_Monitor_StrongPassword_2026 ~* +ping +info +slowlog +client|getname +client|id +client|list +config|get
```

配置 `redis.conf` 引用 ACL 文件：

```conf
aclfile /etc/redis/users.acl
```

> 注意：`>password` 是 ACL 文件中的密码语法。如果通过 shell 执行 `ACL SETUSER`，必须给 `>password` 加引号，否则会被 shell 解释为输出重定向。

### 7.3 应用连接示例

应用应使用 ACL 用户名和密码连接：

```text
redis://app:ReplaceWith_App_StrongPassword_2026@10.10.10.10:16379/0
```

监控工具使用：

```text
redis://monitor:ReplaceWith_Monitor_StrongPassword_2026@10.10.10.10:16379/0
```

### 7.4 redis-cli 安全连接方式

不推荐：

```bash
redis-cli -a ReplaceWith_App_StrongPassword_2026 ping
```

推荐使用环境变量，执行后立即清理：

```bash
export REDISCLI_AUTH='ReplaceWith_App_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user app ping
unset REDISCLI_AUTH
```

也可以使用交互方式：

```bash
redis-cli -h 10.10.10.10 -p 16379 --user app
AUTH app ReplaceWith_App_StrongPassword_2026
```

---

## 8. systemd 服务配置

创建 `/etc/systemd/system/redis.service`：

```bash
cat > /etc/systemd/system/redis.service << 'EOF'
[Unit]
Description=Redis 7.2 Single Instance
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
RuntimeDirectory=redis
RuntimeDirectoryMode=0750
PrivateTmp=true
ProtectHome=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF
```

加载并启用服务：

```bash
systemctl daemon-reload
systemctl enable redis
```

> 如果 Redis 实际安装路径不是 `/data/module/redis7.2/bin/redis-server`，必须修改 `ExecStart`。

---

## 9. 防火墙与网络访问控制

生产 Redis 不允许公网访问，只允许应用服务器、运维跳板机、监控服务器访问。端口规划必须遵循《服务器端口统一规划规范.md》，单体 Redis 标准端口为 `16379`，多实例应在 `16300 ~ 16399` 端口段内分配。

### 9.1 firewalld 示例

将 `10.10.20.0/24` 替换为实际应用服务器网段：

```bash
firewall-cmd --permanent --new-zone=redis
firewall-cmd --permanent --zone=redis --add-source=10.10.20.0/24
firewall-cmd --permanent --zone=redis --add-port=16379/tcp
firewall-cmd --reload
firewall-cmd --zone=redis --list-all
```

如只允许单个应用服务器访问：

```bash
firewall-cmd --permanent --new-zone=redis
firewall-cmd --permanent --zone=redis --add-source=10.10.20.15/32
firewall-cmd --permanent --zone=redis --add-port=16379/tcp
firewall-cmd --reload
```

### 9.2 云安全组要求

如果部署在云主机，安全组必须满足：

| 方向 | 协议 | 端口 | 来源 |
|---|---|---:|---|
| 入站 | TCP | 16379 | 应用服务器内网 IP 或应用网段 |
| 入站 | TCP | 16379 | 监控服务器内网 IP |

禁止：

```text
0.0.0.0/0 -> 16379
```

---

## 10. 启动与基础验证

### 10.1 启动服务

```bash
systemctl start redis
systemctl status redis --no-pager
```

查看日志：

```bash
journalctl -u redis -n 100 --no-pager
less /data/redis/logs/redis.log
```

### 10.2 端口监听验证

```bash
ss -lntp | grep 16379
```

期望只监听 `127.0.0.1` 和指定内网 IP，不应出现公网地址或无约束监听。

### 10.3 认证验证

未认证访问应失败：

```bash
redis-cli -h 10.10.10.10 -p 16379 ping
```

期望返回：

```text
NOAUTH Authentication required.
```

使用应用用户访问：

```bash
export REDISCLI_AUTH='ReplaceWith_App_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user app ping
unset REDISCLI_AUTH
```

期望返回：

```text
PONG
```

### 10.4 ACL 验证

应用用户不应具有管理权限：

```bash
export REDISCLI_AUTH='ReplaceWith_App_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user app CONFIG GET '*'
unset REDISCLI_AUTH
```

期望返回权限不足相关错误。

监控用户应能执行 `INFO`：

```bash
export REDISCLI_AUTH='ReplaceWith_Monitor_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO server
unset REDISCLI_AUTH
```

### 10.5 持久化验证

写入测试 key：

```bash
export REDISCLI_AUTH='ReplaceWith_App_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user app SET prod:check ok EX 300
redis-cli -h 10.10.10.10 -p 16379 --user app GET prod:check
unset REDISCLI_AUTH
```

检查持久化状态：

```bash
export REDISCLI_AUTH='ReplaceWith_Monitor_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO persistence
unset REDISCLI_AUTH
```

关注字段：

```text
loading:0
rdb_last_bgsave_status:ok
aof_enabled:1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
```

### 10.6 资源限制验证

```bash
systemctl show redis -p LimitNOFILE
cat /proc/$(pidof redis-server)/limits | grep 'Max open files'
```

---

## 11. 生产验证清单

上线前必须逐项确认。

### 11.1 OS 检查

| 检查项 | 命令 | 期望 |
|---|---|---|
| THP | `cat /sys/kernel/mm/transparent_hugepage/enabled` | `[never]` |
| overcommit | `sysctl vm.overcommit_memory` | `1` |
| swappiness | `sysctl vm.swappiness` | `0` 或 `1` |
| somaxconn | `sysctl net.core.somaxconn` | `>= 511` |
| 文件句柄 | `systemctl show redis -p LimitNOFILE` | `100032` |

### 11.2 Redis 配置检查

| 检查项 | 期望 |
|---|---|
| 运行用户 | `redis` |
| 监听地址 | 仅 `127.0.0.1` 和内网 IP |
| `protected-mode` | `yes` |
| `aclfile` | 已启用 |
| `default` 用户 | `off` |
| `maxmemory` | 已设置 |
| `maxmemory-policy` | 与业务场景匹配 |
| AOF/RDB | 与数据可靠性要求匹配 |
| 慢日志 | 已启用 |
| `maxclients` | 已设置且小于 `LimitNOFILE` |

### 11.3 安全检查

| 检查项 | 期望 |
|---|---|
| 未认证访问 | 拒绝 |
| 应用用户 | 仅具备读写业务权限 |
| 监控用户 | 仅具备只读监控权限 |
| 公网访问 | 禁止 |
| 安全组/防火墙 | 仅允许指定来源 IP |
| 配置文件权限 | `640 root:redis` |
| ACL 文件权限 | `640 root:redis` |

### 11.4 持久化检查

| 检查项 | 期望 |
|---|---|
| `rdb_last_bgsave_status` | `ok` |
| `aof_last_write_status` | `ok` |
| 数据目录空间 | 充足 |
| 数据目录权限 | `redis` 可写 |
| 重启后数据 | 符合预期 |

---

## 12. 运维检查命令

以下命令建议纳入日常巡检或监控。

### 12.1 服务状态

```bash
systemctl status redis --no-pager
journalctl -u redis -n 100 --no-pager
```

### 12.2 Redis 基础状态

```bash
export REDISCLI_AUTH='ReplaceWith_Monitor_StrongPassword_2026'
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO server
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO clients
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO memory
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO stats
redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO persistence
redis-cli -h 10.10.10.10 -p 16379 --user monitor SLOWLOG LEN
redis-cli -h 10.10.10.10 -p 16379 --user monitor SLOWLOG GET 10
unset REDISCLI_AUTH
```

### 12.3 重点监控指标

| 指标 | 风险说明 |
|---|---|
| `used_memory` | Redis 已使用内存 |
| `used_memory_rss` | 进程实际占用内存 |
| `mem_fragmentation_ratio` | 内存碎片率 |
| `connected_clients` | 当前客户端连接数 |
| `blocked_clients` | 阻塞客户端数量 |
| `rejected_connections` | 被拒绝连接数 |
| `evicted_keys` | 被淘汰 key 数量 |
| `expired_keys` | 过期 key 数量 |
| `keyspace_hits` / `keyspace_misses` | 缓存命中率 |
| `rdb_last_bgsave_status` | RDB 保存状态 |
| `aof_last_write_status` | AOF 写入状态 |
| `latest_fork_usec` | 最近 fork 耗时 |

---

## 13. 变更与回滚建议

### 13.1 配置变更前

备份配置文件：

```bash
cp /etc/redis/redis.conf /etc/redis/redis.conf.bak.$(date +%F-%H%M%S)
cp /etc/redis/users.acl /etc/redis/users.acl.bak.$(date +%F-%H%M%S)
```

检查配置语法：

```bash
/data/module/redis7.2/bin/redis-server /etc/redis/redis.conf --test-memory 2
```

> `--test-memory` 用于内存测试，并不能完整校验所有配置项。配置变更仍需在测试环境验证。

### 13.2 重启策略

单体 Redis 存在单点风险，生产重启前必须确认业务可接受影响窗口。

```bash
systemctl restart redis
systemctl status redis --no-pager
```

### 13.3 回滚策略

如果变更后 Redis 无法启动：

```bash
systemctl stop redis
cp /etc/redis/redis.conf.bak.YYYY-MM-DD-HHMMSS /etc/redis/redis.conf
cp /etc/redis/users.acl.bak.YYYY-MM-DD-HHMMSS /etc/redis/users.acl
systemctl start redis
systemctl status redis --no-pager
```

如果 Redis 启动后应用无法连接，优先检查：

1. `bind` 是否绑定了正确内网 IP；
2. 防火墙或安全组是否允许应用服务器访问；
3. 应用是否使用了 ACL 用户名；
4. `default` 用户禁用后，旧客户端是否仍只传密码不传用户名；
5. ACL 权限是否误删了业务需要的命令。

---

## 后续文档拆分建议

建议 Redis 生产文档拆分为三份：

1. `Redis7.2单体生产级配置部署手册.md`；
2. `Redis7.2主从生产级配置部署手册.md`；
3. `Redis7.2Cluster生产级配置部署手册.md`。

本文仅作为第一份“单体生产级配置部署手册”。主从与 Cluster 文档应复用本文的 OS、用户、目录、安全、systemd 基础规范，再分别补充复制、高可用、故障切换、数据迁移与扩缩容流程。
