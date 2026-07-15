# Redis 7.2 单体生产级安装配置部署手册

> **Author:** 王有政  
> **Encoding:** UTF-8  
> **日期:** 2026-07-15<br>
> **适用系统:** CentOS 7.9 (x86_64)  
> **Redis 版本:** 7.2.14 (BSD-3-Clause)  
> **文档定位:** 单体 Redis 生产级安装配置手册。主从复制、Sentinel 与 Redis Cluster 请拆分为独立文档维护。

---

## 目录

1. [适用范围与部署原则](#1-适用范围与部署原则)
2. [部署规划](#2-部署规划)
3. [部署前检查](#3-部署前检查)
4. [源码包准备与校验](#4-源码包准备与校验)
5. [编译安装 Redis](#5-编译安装-redis)
6. [配置 Redis 环境变量](#6-配置-redis-环境变量)
7. [OS 内核与资源限制配置](#7-os-内核与资源限制配置)
8. [目录、用户与权限](#8-目录用户与权限)
9. [生成生产密码](#9-生成生产密码)
10. [Redis 标准化配置文件写入](#10-redis-标准化配置文件写入)
11. [认证安全配置](#11-认证安全配置)
12. [systemd 服务配置](#12-systemd-服务配置)
13. [网络边界与访问控制](#13-网络边界与访问控制)
14. [启动 Redis](#14-启动-redis)
15. [停止、重启与回滚](#15-停止重启与回滚)

## 1. 适用范围与部署原则

本文用于在 **CentOS 7.9** 上部署 **Redis 7.2.14 单体生产实例**，适用于以下场景：

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
4. 必须启用 Redis 认证，推荐使用 ACL；如客户端或历史系统不支持 ACL，可选择传统 `requirepass` 模式；
5. 必须设置 `maxmemory`，避免系统 OOM；
6. 必须关闭 Transparent Huge Pages；
7. 必须配置持久化策略，或者明确声明该实例是可丢失缓存；
8. 必须由 systemd 托管并设置开机自启；
9. 必须限制访问来源 IP；
10. 必须完成启动验证、持久化验证、安全验证和资源限制验证；
11. 禁止使用 Redis 默认端口 `6379`，必须遵循《服务器端口统一规划规范.md》，默认使用 `16300 ~ 16399` 端口段，单体标准端口为 `16379`。

---

## 2. 部署规划

### 2.1 路径规划


| 类型           | 路径                                          |
| ------------ | ------------------------------------------- |
| 安装根目录        | `/data/module`                              |
| Redis 版本安装目录 | `/data/module/redis7.2.14`                  |
| Redis 稳定软链接  | `/data/module/redis7.2`                     |
| Redis 源码包    | `/data/module/redis-7.2.14.tar.gz`          |
| Redis 解压目录   | `/data/module/redis7.2.14/src/redis-7.2.14` |
| Redis 二进制目录  | `/data/module/redis7.2/bin`                 |
| Redis 配置目录   | `/data/module/redis7.2/conf`                |
| Redis 主配置文件  | `/data/module/redis7.2/conf/redis.conf`     |
| Redis ACL 文件 | `/data/module/redis7.2/conf/users.acl`      |
| Redis 数据目录   | `/data/module/redis7.2/data`                |
| Redis 日志目录   | `/data/module/redis7.2/logs`                |
| Redis PID 目录 | `/data/module/redis7.2/run`                 |
| Redis 备份目录   | `/data/module/redis7.2/backup`              |


说明：

- `/data/module/redis7.2.14` 用于保存 Redis 软件安装结果；
- `/data/module/redis7.2` 是稳定软链接，systemd 和运维命令统一使用该路径；
- `/data` 只是本文示例的数据盘挂载点，实际环境如挂载到 `/data1`、`/mnt/data`、`/u01` 等路径，应按现场挂载点统一替换；
- `/data/module` 用于软件安装目录和版本软链接，不存放 Redis 运行数据；
- `/data/module/redis7.2/conf`、`/data/module/redis7.2/data`、`/data/module/redis7.2/logs`、`/data/module/redis7.2/run`、`/data/module/redis7.2/backup` 分别存放 Redis 配置、数据、日志、运行时文件和备份；
- Redis 单体实例相关文件统一收敛在 `/data/module/redis7.2` 下，便于初始化部署、巡检、迁移和维护。

### 2.2 端口与网络规划


| 项目       | 默认值                    | 说明              |
| -------- | ---------------------- | --------------- |
| Redis 端口 | `16379`                | 单体 Redis 标准端口   |
| 监听地址     | `127.0.0.1` + 本机内网 IP  | 不监听公网地址         |
| 防火墙来源    | 应用服务器、监控服务器、运维跳板机内网 IP | 不允许 `0.0.0.0/0` |


### 2.3 账号规划


| 账号                     | 用途          | 权限原则                 |
| ---------------------- | ----------- | -------------------- |
| Linux 用户 `redis`       | 运行 Redis 进程 | `/sbin/nologin`，不可登录 |
| Redis ACL 用户 `admin`   | 运维管理        | 最高权限，仅限受控运维使用        |
| Redis ACL 用户 `app`     | 应用访问        | 最大化业务权限，排除管理和高危命令   |
| Redis ACL 用户 `monitor` | 监控巡检        | 只读监控命令               |


---

## 3. 部署前检查

### 3.1 服务器资源建议


| 资源   | 生产建议               |
| ---- | ------------------ |
| CPU  | 4 核及以上             |
| 内存   | 8 GB 及以上           |
| 数据盘  | 独立数据盘，建议 SSD       |
| 文件系统 | xfs 或 ext4         |
| 网络   | 仅内网访问              |
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

### 3.2 系统初始化要求

安装 Redis 前，服务器应已完成 `CentOS7.9生产环境初始化手册.md` 中的基础初始化。至少包括：

- 时间同步；
- 主机名与 DNS；
- 数据盘挂载；
- 防火墙基线；
- SSH 安全基线；
- 日志与审计基础配置。

Redis 相关的 THP、sysctl、Swap、文件句柄等配置以本文为准。

---

## 4. 源码包准备与校验

企业内网服务器通常不能直接访问外网，不要在生产服务器上使用 `wget` 或 `git clone` 从 GitHub 拉取源码。

应提前通过企业制品库、堡垒机文件上传、SFTP、SCP、`rz` 等方式，将 `redis-7.2.14.tar.gz` 上传到 `/data/module`。

```bash
mkdir -p /data/module
cd /data/module
ls -lh redis-7.2.14.tar.gz
```

如需使用 `rz` 上传，先确认服务器已有 `lrzsz`：

```bash
rpm -q lrzsz || yum install -y lrzsz
cd /data/module
rz -be
```

上传完成后，建议按企业交付清单校验源码包：

```bash
cd /data/module
sha256sum redis-7.2.14.tar.gz
```

校验值不一致时不要继续安装，应重新获取可信源码包。

---

## 5. 编译安装 Redis

### 5.1 安装编译依赖

```bash
yum install -y gcc make
```

`tcl` 只用于执行 `make test`，生产安装 Redis 不强制要求安装。如企业要求编译后执行测试，可安装：

```bash
yum install -y tcl
```

### 5.2 解压源码包

```bash
cd /data/module
mkdir -p /data/module/redis7.2.14/src
tar -zxvf redis-7.2.14.tar.gz -C /data/module/redis7.2.14/src
cd /data/module/redis7.2.14/src/redis-7.2.14
```

### 5.3 编译

```bash
make -j$(nproc)
```

Redis 默认使用 jemalloc，生产环境推荐保留默认行为。只有在 jemalloc 编译失败且短时间无法处理时，才考虑执行 `make distclean` 后使用 `make MALLOC=libc -j$(nproc)` 重新编译。

如企业流程要求执行测试：

```bash
make test
```

### 5.4 安装到版本目录

```bash
cd /data/module/redis7.2.14/src/redis-7.2.14
make install PREFIX=/data/module/redis7.2.14
```

创建稳定软链接：

```bash
ln -sfn /data/module/redis7.2.14 /data/module/redis7.2
```

### 5.5 验证安装结果

```bash
/data/module/redis7.2/bin/redis-server --version
/data/module/redis7.2/bin/redis-cli --version
ls -lh /data/module/redis7.2/bin
```

示例输出：

```text
Redis server v=7.2.14 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=xxxx
redis-cli 7.2.14
```

---

## 6. 配置 Redis 环境变量

团队统一将自定义环境变量维护在 `/etc/profile.d/myenv.sh`，不要为 Redis 单独创建 `/etc/profile.d/redis.sh`。

编辑统一环境变量文件：

```bash
vim /etc/profile.d/myenv.sh
```

追加以下内容：

```bash
# REDIS_HOME：Redis 当前版本软链接目录，统一指向 /data/module/redis7.2，升级时只切换软链接
export REDIS_HOME=/data/module/redis7.2

# PATH：将 Redis 命令加入命令搜索路径，便于直接执行 redis-cli、redis-server、redis-benchmark 等命令
export PATH=$REDIS_HOME/bin:$PATH
```

保存后加载生效：

```bash
chmod 644 /etc/profile.d/myenv.sh
source /etc/profile.d/myenv.sh
```

验证版本与命令路径：

```bash
redis-server --version
redis-cli --version
which redis-server
which redis-cli
```

期望 `which redis-server` 和 `which redis-cli` 指向 `/data/module/redis7.2/bin/`。

---

## 7. OS 内核与资源限制配置

### 7.1 关闭 Transparent Huge Pages

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

### 7.2 配置 sysctl 参数

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


| 参数                              | 建议值     | 说明                            |
| ------------------------------- | ------- | ----------------------------- |
| `vm.overcommit_memory`          | `1`     | 避免 Redis fork 子进程时因内存分配策略导致失败 |
| `vm.swappiness`                 | `1`     | 尽量避免 Redis 内存页被换出             |
| `net.core.somaxconn`            | `65535` | 提高 TCP accept 队列上限            |
| `net.ipv4.tcp_max_syn_backlog`  | `65535` | 提高 SYN 队列上限                   |
| `net.ipv4.tcp_keepalive_time`   | `300`   | TCP keepalive 探测时间            |
| `net.ipv4.tcp_keepalive_intvl`  | `30`    | TCP keepalive 探测间隔            |
| `net.ipv4.tcp_keepalive_probes` | `5`     | TCP keepalive 探测次数            |


验证：

```bash
sysctl vm.overcommit_memory
sysctl vm.swappiness
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
```

### 7.3 Swap 策略

保留 Swap 作为内存耗尽时的系统兜底，同时把使用倾向降到最低：

```bash
sysctl vm.swappiness=1
```

### 7.4 文件句柄数限制

Redis 每个客户端连接至少占用一个文件描述符。生产环境必须提高文件句柄限制。

```bash
cat > /etc/security/limits.d/99-redis.conf << 'EOF'
redis soft nofile 100032
redis hard nofile 100032
EOF
```

systemd 服务中也需要设置 `LimitNOFILE=100032`，见 [12. systemd 服务配置](#12-systemd-服务配置)。

---

## 8. 目录、用户与权限

### 8.1 创建 redis 用户

```bash
id redis || useradd -r -s /sbin/nologin redis
```

### 8.2 创建运行目录

```bash
mkdir -p /data/module/redis7.2/conf
mkdir -p /data/module/redis7.2/data
mkdir -p /data/module/redis7.2/logs
mkdir -p /data/module/redis7.2/run
mkdir -p /data/module/redis7.2/backup
```

`/data/module/redis7.2/run` 为 Redis 运行时目录，统一放在 Redis 应用目录下，便于单应用维护和巡检。

目录用途：


| 目录                             | 用途                    |
| ------------------------------ | --------------------- |
| `/data/module/redis7.2/conf`   | Redis 配置文件与 ACL 文件    |
| `/data/module/redis7.2/data`   | RDB、AOF、临时 rewrite 文件 |
| `/data/module/redis7.2/logs`   | Redis 日志              |
| `/data/module/redis7.2/run`    | PID 文件                |
| `/data/module/redis7.2/backup` | 备份文件暂存目录              |


### 8.3 设置权限

```bash
chown -R redis:redis /data/module/redis7.2/data /data/module/redis7.2/logs /data/module/redis7.2/run /data/module/redis7.2/backup
chown root:redis /data/module/redis7.2/conf
chmod 750 /data/module/redis7.2/conf
chmod 750 /data/module/redis7.2/data /data/module/redis7.2/logs /data/module/redis7.2/run /data/module/redis7.2/backup
```

`/data/module/redis7.2/run` 存放 Redis PID/socket 等运行时文件，统一收敛在 Redis 应用目录下。

配置文件建议归属 `root:redis`，Redis 进程只读配置，不应拥有修改配置文件的权限。

---

## 9. 生成生产密码

Redis 安装配置前先生成强密码，并用生成结果替换后文所有示例密码。

### 9.1 生成复杂密码

如果企业密码规范要求必须包含大小写字母、数字和特殊字符，推荐使用以下方式生成：

```bash
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:,.~!#$^*-' < /dev/urandom | head -c 48; echo
```

该命令会生成随机复杂密码，字符集刻意避开了空格、单引号、双引号、反斜杠、反引号、分号、管道符等容易影响配置文件或命令行解析的字符。

## 10. Redis 标准化配置文件写入

Redis 支持在主配置文件中使用 `include` 引入额外配置文件。为兼顾“保留官方默认配置参考”和“首次部署快速标准化”，本文采用以下方式：

- `/data/module/redis7.2/conf/redis.conf`：主配置文件，来自官方默认配置，只在末尾追加 `include`；
- `/data/module/redis7.2/conf/redis-production.conf`：生产标准化配置文件，完整写入生产参数；
- `/data/module/redis7.2/conf/users.acl`：Redis ACL 权限文件，仅在选择 ACL 认证模式时使用。

`include` 必须追加在 `redis.conf` 末尾，使 `redis-production.conf` 中的同名配置覆盖官方默认配置。首次部署时主要维护 `redis-production.conf`，不需要逐项搜索修改官方默认配置文件。

### 10.1 准备主配置文件并追加 include

```bash
cp /data/module/redis7.2.14/src/redis-7.2.14/redis.conf /data/module/redis7.2/conf/redis.conf

cat >> /data/module/redis7.2/conf/redis.conf << 'EOF'

# Load production overrides. Keep this include at the end of redis.conf.
include /data/module/redis7.2/conf/redis-production.conf
EOF

chown root:redis /data/module/redis7.2/conf/redis.conf
chmod 640 /data/module/redis7.2/conf/redis.conf
```

### 10.2 写入生产标准化配置文件

写入前先设置现场变量。必须将 `REDIS_BIND_IP` 替换为本机实际内网 IP（在服务器执行ifconfig查看)，`REDIS_MAXMEMORY` 按服务器内存和业务容量评估后调整。

```bash
REDIS_BIND_IP="10.10.10.10"
REDIS_PORT="16379"
REDIS_MAXMEMORY="8gb"

cat > /data/module/redis7.2/conf/redis-production.conf << EOF
# Redis 7.2 single instance production config
# Author: 王有政

# ==== basic ====
daemonize no
supervised no
pidfile /data/module/redis7.2/run/redis.pid
loglevel notice
logfile /data/module/redis7.2/logs/redis.log
dir /data/module/redis7.2/data
databases 16
always-show-logo no

# ==== network ====
bind 127.0.0.1 ${REDIS_BIND_IP}
protected-mode yes
port ${REDIS_PORT}
tcp-backlog 511
timeout 0
# Java 服务连接池场景不建议由 Redis 主动关闭正常空闲连接，异常死连接由 TCP keepalive 探测
tcp-keepalive 300

# ==== memory ====
maxmemory ${REDIS_MAXMEMORY}
maxmemory-policy allkeys-lfu
maxmemory-samples 10

# ==== persistence: RDB ====
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb

# ==== persistence: AOF ====
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 512mb
aof-use-rdb-preamble yes

# ==== performance ====
io-threads 4
io-threads-do-reads no
hz 10
dynamic-hz yes
activerehashing yes
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

# ==== slowlog and latency ====
slowlog-log-slower-than 10000
slowlog-max-len 1024
latency-monitor-threshold 100

# ==== clients ====
maxclients 50000
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# ==== security ====
aclfile /data/module/redis7.2/conf/users.acl
EOF

chown root:redis /data/module/redis7.2/conf/redis-production.conf
chmod 640 /data/module/redis7.2/conf/redis-production.conf
```

### 10.3 创建 ACL 文件

```bash
touch /data/module/redis7.2/conf/users.acl
chown root:redis /data/module/redis7.2/conf/users.acl
chmod 640 /data/module/redis7.2/conf/users.acl
```

检查配置文件关键路径：

```bash
grep -E '^(include|bind|port|maxmemory|dir|logfile|pidfile|aclfile) ' \
  /data/module/redis7.2/conf/redis.conf \
  /data/module/redis7.2/conf/redis-production.conf
```

> 注意：如果 Redis 仅供本机应用访问，可将 `bind 127.0.0.1 ${REDIS_BIND_IP}` 改为 `bind 127.0.0.1`。生产环境禁止使用 `bind 0.0.0.0` 作为默认配置。

---

## 11. 认证安全配置

生产环境统一使用 ACL 认证，禁用 `default` 用户，并按管理、应用和监控用途拆分账号。

### 11.1 ACL 认证配置

#### 11.1.1 ACL 设计原则

生产环境建议：

1. 禁用 `default` 用户；
2. 创建 `admin` 管理用户并授予最高权限，仅用于受控运维、ACL 维护和故障处理；
3. 创建 `app` 应用用户，采用“最大化业务兼容、排除管理和高危命令”的权限模型；
4. 创建 `monitor` 只读监控用户，仅用于监控采集和巡检；
5. 不在命令行中直接暴露密码；
6. 不使用 `redis-cli -a 明文密码` 作为生产执行规范；
7. 密码必须由密码管理系统生成和保存。

账号使用边界：

- `admin`：只允许运维人员、自动化运维任务或受控堡垒机使用，不配置到业务应用；
- `app`：业务应用专用账号，尽量兼容 Java/Spring/Lettuce/Redisson 常见能力，但不允许执行管理类和高危命令；
- `monitor`：监控巡检专用账号，只允许读取运行状态，不允许读写业务数据。

#### 11.1.2 Redis 配置

使用 ACL 认证时，`/data/module/redis7.2/conf/redis-production.conf` 中保留：

```conf
aclfile /data/module/redis7.2/conf/users.acl
```

不要再配置：

```conf
requirepass ReplaceWith_StrongPassword
```

#### 11.1.3 创建 ACL 文件

ACL 文件只保存密码的 SHA-256 哈希，原始密码保存到企业密码管理系统：

```bash
REDIS_ADMIN_PASSWORD='ReplaceWith_Admin_StrongPassword'
REDIS_APP_PASSWORD='ReplaceWith_App_StrongPassword'
REDIS_MONITOR_PASSWORD='ReplaceWith_Monitor_StrongPassword'

REDIS_ADMIN_PASSWORD_HASH=$(printf '%s' "$REDIS_ADMIN_PASSWORD" | sha256sum | cut -d ' ' -f 1)
REDIS_APP_PASSWORD_HASH=$(printf '%s' "$REDIS_APP_PASSWORD" | sha256sum | cut -d ' ' -f 1)
REDIS_MONITOR_PASSWORD_HASH=$(printf '%s' "$REDIS_MONITOR_PASSWORD" | sha256sum | cut -d ' ' -f 1)

cat > /data/module/redis7.2/conf/users.acl << EOF
user default off
user admin on #${REDIS_ADMIN_PASSWORD_HASH} ~* &* +@all
user app on #${REDIS_APP_PASSWORD_HASH} ~* &* +@all -@admin -@dangerous -acl -config -shutdown -debug -module -flushdb -flushall -keys -monitor -save -bgsave -bgrewriteaof +info
user monitor on #${REDIS_MONITOR_PASSWORD_HASH} ~* +ping +info +slowlog +client|getname +client|id +client|list +config|get
EOF
```

设置权限：

```bash
chown root:redis /data/module/redis7.2/conf/users.acl
chmod 640 /data/module/redis7.2/conf/users.acl

unset REDIS_ADMIN_PASSWORD REDIS_APP_PASSWORD REDIS_MONITOR_PASSWORD
unset REDIS_ADMIN_PASSWORD_HASH REDIS_APP_PASSWORD_HASH REDIS_MONITOR_PASSWORD_HASH
```

#### 11.1.4 ACL 连接示例

应用连接：

```text
redis://app:<APP_STRONG_PASSWORD>@10.10.10.10:16379/0
```

管理连接：

```text
redis://admin:<ADMIN_STRONG_PASSWORD>@10.10.10.10:16379/0
```

监控工具连接：

```text
redis://monitor:<MONITOR_STRONG_PASSWORD>@10.10.10.10:16379/0
```

`app` 用户包含 `+@write` 权限，不是只读账号；`monitor` 用户才是监控只读账号。如果图形客户端提示 `readonly mode` 或 `Unable to execute write command`，优先检查客户端连接配置中是否勾选了 `Readonly`、`Read only`、`只读模式` 等选项。Redis ACL 权限不足时通常返回 `NOPERM`，而不是客户端只读模式提示。

`redis-cli` 推荐使用环境变量，执行后立即清理：

```bash
export REDISCLI_AUTH='<APP_STRONG_PASSWORD>'
redis-cli -h 10.10.10.10 -p 16379 --user app ping
unset REDISCLI_AUTH
```

也可以使用交互方式：

```bash
redis-cli -h 10.10.10.10 -p 16379 --user app
AUTH app <APP_STRONG_PASSWORD>
```

## 12. systemd 服务配置

创建 `/etc/systemd/system/redis.service`：

```bash
cat > /etc/systemd/system/redis.service << 'EOF'
[Unit]
Description=Redis 7.2 Single Instance
Documentation=https://redis.io/docs/
After=network-online.target disable-thp.service
Wants=network-online.target

[Service]
Type=simple
User=redis
Group=redis
ExecStart=/data/module/redis7.2/bin/redis-server /data/module/redis7.2/conf/redis.conf --daemonize no
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

加载并启用服务：

```bash
systemctl daemon-reload
systemctl enable redis
```

如果 Redis 实际安装路径不是 `/data/module/redis7.2/bin/redis-server`，必须修改 `ExecStart`。

---

## 13. 网络边界与访问控制

本文不维护本机 `firewalld` 规则，主机防火墙基线由《CentOS7.9生产环境初始化手册.md》统一规定。

Redis 单体标准端口为 `16379`，仅允许应用服务器、运维跳板机、监控服务器访问；禁止公网直接访问。

部署前需要确认：

- 端口仅对可信内网开放；
- 访问来源由外层防火墙、安全组、ACL、VPN、堡垒机或统一网络策略控制；
- 禁止 `0.0.0.0/0` 访问 Redis 端口；
- 端口、来源、用途应登记到资产台账或 CMDB。

---

## 14. 启动 Redis

### 14.1 启动服务

```bash
systemctl start redis
systemctl status redis --no-pager
```

查看日志：

```bash
journalctl -u redis -n 100 --no-pager
less /data/module/redis7.2/logs/redis.log
```

## 15. 停止、重启与回滚

### 15.1 停止 Redis

优先使用 systemd：

```bash
systemctl stop redis
systemctl status redis --no-pager
```

不建议在生产规范中直接使用明文密码执行 `shutdown`。

### 15.2 重启 Redis

单体 Redis 存在单点风险，生产重启前必须确认业务可接受影响窗口。

```bash
systemctl restart redis
systemctl status redis --no-pager
```

### 15.3 配置变更前

备份配置文件：

```bash
cp /data/module/redis7.2/conf/redis.conf /data/module/redis7.2/conf/redis.conf.bak.$(date +%F-%H%M%S)
cp /data/module/redis7.2/conf/users.acl /data/module/redis7.2/conf/users.acl.bak.$(date +%F-%H%M%S)
```

配置变更仍需在测试环境验证。Redis 没有等价于 Nginx `nginx -t` 的完整配置检查命令，不要把 `--test-memory` 误当成配置语法检查。

### 15.4 回滚配置

如果变更后 Redis 无法启动：

```bash
systemctl stop redis
cp /data/module/redis7.2/conf/redis.conf.bak.YYYY-MM-DD-HHMMSS /data/module/redis7.2/conf/redis.conf
cp /data/module/redis7.2/conf/users.acl.bak.YYYY-MM-DD-HHMMSS /data/module/redis7.2/conf/users.acl
chown root:redis /data/module/redis7.2/conf/redis.conf /data/module/redis7.2/conf/users.acl
chmod 640 /data/module/redis7.2/conf/redis.conf /data/module/redis7.2/conf/users.acl
systemctl start redis
systemctl status redis --no-pager
```

如果 Redis 启动后应用无法连接，优先检查：

1. `bind` 是否绑定了正确内网 IP；
2. 防火墙或安全组是否允许应用服务器访问；
3. 应用是否使用了 ACL 用户名；
4. `default` 用户禁用后，旧客户端是否仍只传密码不传用户名；
5. ACL 权限是否误删了业务需要的命令。

### 15.5 版本回滚

如果升级后需要回滚 Redis 二进制版本，应先确认旧版本安装目录仍存在。例如：

```bash
ls -ld /data/module/redis7.2.14
ls -ld /data/module/redis7.2.OLD_VERSION
```

停止服务后切换软链接：

```bash
systemctl stop redis
ln -sfn /data/module/redis7.2.OLD_VERSION /data/module/redis7.2
systemctl start redis
systemctl status redis --no-pager
```

版本回滚前必须评估 RDB/AOF 文件兼容性，不要在未验证数据文件兼容的情况下跨大版本回滚。

---
