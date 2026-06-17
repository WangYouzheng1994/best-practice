# Redis 7.2 单体生产级安装配置部署手册

> **Author:** 王有政  
> **Encoding:** UTF-8  
> **日期:** 2026-06-16  
> **适用系统:** CentOS 7.9 (x86_64)  
> **Redis 版本:** 7.2.14 (BSD-3-Clause)  
> **文档定位:** 单体 Redis 生产级从安装到配置、启动、验证、运维与回滚的一站式部署手册。主从复制、Sentinel 与 Redis Cluster 请拆分为独立文档维护。

---

## 目录

1. [适用范围与部署原则](#1-适用范围与部署原则)
2. [部署规划](#2-部署规划)
3. [部署前检查](#3-部署前检查)
4. [源码包准备与校验](#4-源码包准备与校验)
5. [编译安装 Redis](#5-编译安装-redis)
6. [OS 内核与资源限制配置](#6-os-内核与资源限制配置)
7. [目录、用户与权限](#7-目录用户与权限)
8. [生成生产密码](#8-生成生产密码)
9. [Redis 标准化配置文件写入](#9-redis-标准化配置文件写入)
10. [单体 Redis 核心配置说明](#10-单体-redis-核心配置说明)
11. [认证安全配置](#11-认证安全配置)
12. [systemd 服务配置](#12-systemd-服务配置)
13. [网络边界与访问控制](#13-网络边界与访问控制)
14. [启动与基础验证](#14-启动与基础验证)
15. [生产验证清单](#15-生产验证清单)
16. [运维检查命令](#16-运维检查命令)
17. [停止、重启与回滚](#17-停止重启与回滚)
18. [常见问题](#18-常见问题)

---

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

## 6. OS 内核与资源限制配置

### 6.1 关闭 Transparent Huge Pages

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

### 6.2 配置 sysctl 参数

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

### 6.3 Swap 策略

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

### 6.4 文件句柄数限制

Redis 每个客户端连接至少占用一个文件描述符。生产环境必须提高文件句柄限制。

```bash
cat > /etc/security/limits.d/99-redis.conf << 'EOF'
redis soft nofile 100032
redis hard nofile 100032
EOF
```

systemd 服务中也需要设置 `LimitNOFILE=100032`，见 [12. systemd 服务配置](#12-systemd-服务配置)。

---

## 7. 目录、用户与权限

### 7.1 创建 redis 用户

```bash
id redis || useradd -r -s /sbin/nologin redis
```

### 7.2 创建运行目录

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


### 7.3 设置权限

```bash
chown -R redis:redis /data/module/redis7.2/data /data/module/redis7.2/logs /data/module/redis7.2/run /data/module/redis7.2/backup
chown root:redis /data/module/redis7.2/conf
chmod 750 /data/module/redis7.2/conf
chmod 750 /data/module/redis7.2/data /data/module/redis7.2/logs /data/module/redis7.2/run /data/module/redis7.2/backup
```

`/data/module/redis7.2/run` 存放 Redis PID/socket 等运行时文件，统一收敛在 Redis 应用目录下。

配置文件建议归属 `root:redis`，Redis 进程只读配置，不应拥有修改配置文件的权限。

---

## 8. 生成生产密码

Redis 安装配置前先生成强密码，并用生成结果替换后文所有示例密码。

### 8.1 方案 A：生成包含特殊字符的复杂密码

如果企业密码规范要求必须包含大小写字母、数字和特殊字符，推荐使用以下方式生成：

```bash
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:,.~!#$^*-' < /dev/urandom | head -c 48; echo
```

该命令会生成随机复杂密码，字符集刻意避开了空格、单引号、双引号、反斜杠、反引号、分号、管道符等容易影响配置文件或命令行解析的字符。

### 8.2 方案 B：生成十六进制随机密码

```bash
openssl rand -hex 32
```

该方式生成 64 位十六进制字符，随机强度高，且不包含容易被 Shell 误解析的特殊字符。但它只包含 `0-9a-f`，不满足部分企业对“必须包含大小写字母、数字、特殊字符”的密码复杂度要求。

### 8.3 方案 C：没有 OpenSSL 时生成普通随机密码

```bash
tr -dc 'A-Za-z0-9_@%+=:,.~-' < /dev/urandom | head -c 48; echo
```

密码要求：不少于 32 字符，必须随机生成。不要使用人工编造的 `P@ssw0rd`、`Redis@123`、`Admin@123456`、姓名、项目名、公司名、手机号、生日、常见单词等可猜测内容。

**推荐：** 优先使用方案 A。如果现场更重视命令行和配置文件转义安全，且企业密码策略不强制要求特殊字符，可使用方案 B。真实密码不要写入文档、提交到 Git 仓库或发送到群聊，应保存到企业密码管理系统或受控密钥管理平台。

---

## 9. Redis 标准化配置文件写入

Redis 支持在主配置文件中使用 `include` 引入额外配置文件。为兼顾“保留官方默认配置参考”和“首次部署快速标准化”，本文采用以下方式：

- `/data/module/redis7.2/conf/redis.conf`：主配置文件，来自官方默认配置，只在末尾追加 `include`；
- `/data/module/redis7.2/conf/redis-production.conf`：生产标准化配置文件，完整写入生产参数；
- `/data/module/redis7.2/conf/users.acl`：Redis ACL 权限文件，仅在选择 ACL 认证模式时使用。

`include` 必须追加在 `redis.conf` 末尾，使 `redis-production.conf` 中的同名配置覆盖官方默认配置。首次部署时主要维护 `redis-production.conf`，不需要逐项搜索修改官方默认配置文件。

### 9.1 准备主配置文件并追加 include

```bash
cp /data/module/redis7.2.14/src/redis-7.2.14/redis.conf /data/module/redis7.2/conf/redis.conf

cat >> /data/module/redis7.2/conf/redis.conf << 'EOF'

# Load production overrides. Keep this include at the end of redis.conf.
include /data/module/redis7.2/conf/redis-production.conf
EOF

chown root:redis /data/module/redis7.2/conf/redis.conf
chmod 640 /data/module/redis7.2/conf/redis.conf
```

### 9.2 写入生产标准化配置文件

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
timeout 300
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

### 9.3 创建 ACL 文件

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

## 10. 单体 Redis 核心配置说明

本章只解释第 9 章标准配置文件中的核心参数含义和调整依据。首次部署时优先直接使用第 9 章完整配置写入，不需要逐项手工修改官方默认配置文件。

### 10.1 基础运行配置

```conf
daemonize no
supervised no
pidfile /data/module/redis7.2/run/redis.pid
logfile /data/module/redis7.2/logs/redis.log
dir /data/module/redis7.2/data
```

说明：

- `daemonize no`：由 systemd 托管前台进程，不让 Redis 自行后台化；
- `supervised no`：不依赖 Redis 的 systemd notify 支持，避免源码编译未启用 `libsystemd` 时服务卡在 `activating`；
- `pidfile /data/module/redis7.2/run/redis.pid`：PID 属于运行时文件，放在 `/run`；
- `logfile /data/module/redis7.2/logs/redis.log`：日志进入统一日志目录；
- `dir /data/module/redis7.2/data`：RDB、AOF、rewrite 临时文件进入统一数据目录。

### 10.2 网络配置

```conf
bind 127.0.0.1 10.10.10.10
protected-mode yes
port 16379
tcp-backlog 511
timeout 300
tcp-keepalive 300
```

生产环境禁止默认绑定 `0.0.0.0`。应只绑定本机回环地址和 Redis 服务内网 IP。

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

### 10.3 内存上限配置

Redis 必须设置 `maxmemory`，避免 Redis 挤占系统内存导致 OOM。

```conf
maxmemory 8gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10
```

`maxmemory` 建议：


| 服务器内存 | 推荐 `maxmemory` |
| ----- | -------------- |
| 8 GB  | 4 GB ~ 5 GB    |
| 16 GB | 8 GB ~ 10 GB   |
| 32 GB | 18 GB ~ 22 GB  |
| 64 GB | 38 GB ~ 45 GB  |


需要为以下内容预留内存：

- 操作系统；
- Redis fork 时的 Copy-On-Write；
- AOF rewrite；
- RDB 快照；
- 客户端输出缓冲区；
- jemalloc 内存碎片；
- 监控 Agent 和系统服务。

### 10.4 淘汰策略选择


| 策略             | 适用场景                | 风险                     |
| -------------- | ------------------- | ---------------------- |
| `allkeys-lfu`  | 通用缓存、热点数据保护         | 所有 key 都可能被淘汰          |
| `allkeys-lru`  | 通用缓存                | 不如 LFU 保护热点数据          |
| `volatile-lfu` | 只有设置 TTL 的 key 允许淘汰 | 未设置 TTL 的 key 可能导致 OOM |
| `volatile-lru` | 兼容传统缓存策略            | 未设置 TTL 的 key 可能导致 OOM |
| `noeviction`   | 不能丢数据的状态类场景         | 写入达到上限后报错              |


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

### 10.5 持久化配置

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
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
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

### 10.6 性能相关配置

```conf
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

### 10.7 慢日志与延迟监控

```conf
slowlog-log-slower-than 10000
slowlog-max-len 1024
latency-monitor-threshold 100
```

说明：

- `slowlog-log-slower-than 10000` 表示记录执行超过 10 ms 的命令；
- 低延迟业务可调整为 `5000`；
- 生产环境不建议设置为负数禁用慢日志。

### 10.8 客户端连接管理

```conf
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

### 10.9 ACL 文件引用

```conf
aclfile /data/module/redis7.2/conf/users.acl
```

ACL 账号和权限规则在第 11 章维护。

### 10.10 禁用危险命令的策略

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

## 11. 认证安全配置

Redis 7.2 支持两类认证方式：

| 方案 | 认证方式 | 客户端连接 | 权限控制能力 | 推荐场景 |
|---|---|---|---|---|
| 方案 A：ACL 认证 | `aclfile` + ACL 用户 | 需要用户名和密码 | 可区分应用、监控、运维用户 | 推荐，适合新系统和生产规范化部署 |
| 方案 B：传统密码认证 | `requirepass` | 只需要密码 | 所有客户端使用 `default` 用户权限 | 兼容旧客户端、历史系统或不支持 ACL 的工具 |

二者选择其一即可，不要同时启用 `aclfile` 和 `requirepass`。如果已经启用 ACL 并禁用了 `default` 用户，再单独配置 `requirepass` 不能解决旧客户端只填密码的问题。

**推荐：** 新部署优先使用方案 A。只有在客户端、框架或历史系统确实不支持 Redis ACL 用户名时，才选择方案 B。

### 11.1 方案 A：ACL 认证

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
requirepass wyz123!@#
```

#### 11.1.3 创建 ACL 文件

Redis ACL 文件支持两种密码写法：

| 写法 | 示例 | 说明 | 适用建议 |
|---|---|---|---|
| 明文写法 | `>原始密码` | Redis 启动后会按该密码认证，文件中可直接看到原始密码 | 本文主流程采用，简单直观，便于首次部署和排障 |
| 哈希写法 | `#SHA256哈希值` | ACL 文件中不保存原始密码，只保存密码的 SHA-256 哈希 | 可选增强方案，适合对配置文件泄露风险更敏感的环境 |

无论 ACL 文件中使用 `>原始密码` 还是 `#SHA256哈希值`，客户端连接时都使用原始密码，不使用哈希值。

本文主流程使用明文密码写入 ACL 文件：

```bash
cat > /data/module/redis7.2/conf/users.acl << 'EOF'
user default off
user admin on >wyz123!@# ~* &* +@all
user app on >wyz123!@# ~* &* +@all -@admin -@dangerous -acl -config -shutdown -debug -module -flushdb -flushall -keys -monitor -save -bgsave -bgrewriteaof
user monitor on >wyz123!@# ~* +ping +info +slowlog +client|getname +client|id +client|list +config|get
EOF
```

设置权限：

```bash
chown root:redis /data/module/redis7.2/conf/users.acl
chmod 640 /data/module/redis7.2/conf/users.acl
```

`admin` 用户拥有最高权限，仅用于受控运维操作，不应配置到业务应用。`app` 用户是业务应用通用账号，采用最大化兼容策略：先授予 `+@all`，再移除管理类和高危命令，默认覆盖 Java/Spring/Lettuce/Redisson 常见业务能力，包括普通读写、事务、Lua 脚本、Pub/Sub、Stream、GEO、Bitmap、HyperLogLog 等。`~*` 表示允许访问所有 key，`&*` 表示允许访问所有 Pub/Sub channel。该账号不授予 ACL 管理、配置修改、停服、调试、模块加载、清库、全量遍历、监控连接抓取和手工持久化触发等能力。

注意：`>password` 是 ACL 文件中的密码语法。如果通过 shell 执行 `ACL SETUSER`，必须给 `>password` 加引号，否则会被 shell 解释为输出重定向。

#### 11.1.4 可选：使用 SHA-256 哈希写法保存 ACL 密码

如果希望 ACL 文件中不直接保存原始密码，可以改用 SHA-256 哈希写法。该方式不是本文部署的必须动作，只是可选增强方案。

先生成哈希值：

```bash
REDIS_ADMIN_PASSWORD='wyz123!@#'
REDIS_APP_PASSWORD='wyz123!@#'
REDIS_MONITOR_PASSWORD='wyz123!@#'

REDIS_ADMIN_PASSWORD_HASH=$(printf '%s' "$REDIS_ADMIN_PASSWORD" | sha256sum | cut -d ' ' -f 1)
REDIS_APP_PASSWORD_HASH=$(printf '%s' "$REDIS_APP_PASSWORD" | sha256sum | cut -d ' ' -f 1)
REDIS_MONITOR_PASSWORD_HASH=$(printf '%s' "$REDIS_MONITOR_PASSWORD" | sha256sum | cut -d ' ' -f 1)

printf 'admin hash: %s\n' "$REDIS_ADMIN_PASSWORD_HASH"
printf 'app hash: %s\n' "$REDIS_APP_PASSWORD_HASH"
printf 'monitor hash: %s\n' "$REDIS_MONITOR_PASSWORD_HASH"
```

再写入 ACL 文件：

```bash
cat > /data/module/redis7.2/conf/users.acl << EOF
user default off
user admin on #${REDIS_ADMIN_PASSWORD_HASH} ~* &* +@all
user app on #${REDIS_APP_PASSWORD_HASH} ~* &* +@all -@admin -@dangerous -acl -config -shutdown -debug -module -flushdb -flushall -keys -monitor -save -bgsave -bgrewriteaof
user monitor on #${REDIS_MONITOR_PASSWORD_HASH} ~* +ping +info +slowlog +client|getname +client|id +client|list +config|get
EOF
```

设置权限：

```bash
chown root:redis /data/module/redis7.2/conf/users.acl
chmod 640 /data/module/redis7.2/conf/users.acl
```

#### 11.1.5 ACL 连接示例

应用连接：

```text
redis://app:wyz123!@#@10.10.10.10:16379/0
```

管理连接：

```text
redis://admin:wyz123!@#@10.10.10.10:16379/0
```

监控工具连接：

```text
redis://monitor:wyz123!@#@10.10.10.10:16379/0
```

`app` 用户包含 `+@write` 权限，不是只读账号；`monitor` 用户才是监控只读账号。如果图形客户端提示 `readonly mode` 或 `Unable to execute write command`，优先检查客户端连接配置中是否勾选了 `Readonly`、`Read only`、`只读模式` 等选项。Redis ACL 权限不足时通常返回 `NOPERM`，而不是客户端只读模式提示。

`redis-cli` 推荐使用环境变量，执行后立即清理：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user app ping
unset REDISCLI_AUTH
```

也可以使用交互方式：

```bash
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user app
AUTH app wyz123!@#
```

### 11.2 方案 B：传统 `requirepass` 认证

传统 `requirepass` 认证只需要密码，不需要用户名。它兼容旧客户端和只支持 Redis 5 以前认证方式的工具，但无法区分应用用户、监控用户和运维用户。

#### 11.2.1 Redis 配置

使用传统认证时，`/data/module/redis7.2/conf/redis-production.conf` 中不要启用：

```conf
aclfile /data/module/redis7.2/conf/users.acl
```

改为配置：

```conf
requirepass wyz123!@#
```

完整安全配置段示例：

```conf
# ==== security ====
requirepass wyz123!@#
```

如果之前已经创建并启用了 ACL 文件，切换到传统认证时应从 `redis-production.conf` 中移除或注释 `aclfile`，然后重启 Redis。

#### 11.2.2 传统认证连接示例

应用连接：

```text
redis://:wyz123!@#@10.10.10.10:16379/0
```

`redis-cli` 推荐使用环境变量，执行后立即清理：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 ping
unset REDISCLI_AUTH
```

不推荐：

```bash
redis-cli -h 10.10.10.10 -p 16379 -a wyz123!@# ping
```

### 11.3 认证方案切换后的生效方式

无论选择 ACL 认证还是传统 `requirepass` 认证，修改认证配置后都需要重启 Redis：

```bash
systemctl restart redis
systemctl status redis --no-pager
```

验证端口监听：

```bash
ss -lntp | grep 16379
```

验证认证：

```bash
# ACL 认证
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 127.0.0.1 -p 16379 --user app ping
unset REDISCLI_AUTH

# 传统 requirepass 认证
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 127.0.0.1 -p 16379 ping
unset REDISCLI_AUTH
```

---

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

## 14. 启动与基础验证

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

### 14.2 端口监听验证

```bash
ss -lntp | grep 16379
```

期望只监听 `127.0.0.1` 和指定内网 IP，不应出现公网地址或无约束监听。

### 14.3 认证验证

未认证访问应失败：

```bash
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 ping
```

期望返回：

```text
NOAUTH Authentication required.
```

使用应用用户访问：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user app ping
unset REDISCLI_AUTH
```

期望返回：

```text
PONG
```

### 14.4 ACL 验证

应用用户不应具有管理权限：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user app CONFIG GET '*'
unset REDISCLI_AUTH
```

期望返回权限不足相关错误。

监控用户应能执行 `INFO`：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO server
unset REDISCLI_AUTH
```

### 14.5 持久化验证

写入测试 key：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user app SET prod:check ok EX 300
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user app GET prod:check
unset REDISCLI_AUTH
```

检查持久化状态：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO persistence
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

### 14.6 资源限制验证

```bash
systemctl show redis -p LimitNOFILE
cat /proc/$(pidof redis-server)/limits | grep 'Max open files'
```

---

## 15. 生产验证清单

上线前必须逐项确认。

### 15.1 安装检查


| 检查项          | 命令                                                 | 期望                            |
| ------------ | -------------------------------------------------- | ----------------------------- |
| Redis 版本     | `/data/module/redis7.2/bin/redis-server --version` | `7.2.14`                      |
| redis-cli 版本 | `/data/module/redis7.2/bin/redis-cli --version`    | `7.2.14`                      |
| 稳定软链接        | `ls -l /data/module/redis7.2`                      | 指向 `/data/module/redis7.2.14` |
| 运行用户         | `id redis`                                         | 用户存在且不可登录                     |


### 15.2 OS 检查


| 检查项        | 命令                                                | 期望        |
| ---------- | ------------------------------------------------- | --------- |
| THP        | `cat /sys/kernel/mm/transparent_hugepage/enabled` | `[never]` |
| overcommit | `sysctl vm.overcommit_memory`                     | `1`       |
| swappiness | `sysctl vm.swappiness`                            | `0` 或 `1` |
| somaxconn  | `sysctl net.core.somaxconn`                       | `>= 511`  |
| 文件句柄       | `systemctl show redis -p LimitNOFILE`             | `100032`  |


### 15.3 Redis 配置检查


| 检查项                | 期望                   |
| ------------------ | -------------------- |
| 运行用户               | `redis`              |
| 监听地址               | 仅 `127.0.0.1` 和内网 IP |
| `protected-mode`   | `yes`                |
| `aclfile`          | 已启用                  |
| `default` 用户       | `off`                |
| `maxmemory`        | 已设置                  |
| `maxmemory-policy` | 与业务场景匹配              |
| AOF/RDB            | 与数据可靠性要求匹配           |
| 慢日志                | 已启用                  |
| `maxclients`       | 已设置且小于 `LimitNOFILE` |


### 15.4 安全检查


| 检查项      | 期望               |
| -------- | ---------------- |
| 未认证访问    | 拒绝               |
| 应用用户     | 仅具备读写业务权限        |
| 监控用户     | 仅具备只读监控权限        |
| 公网访问     | 禁止               |
| 安全组/防火墙  | 仅允许指定来源 IP       |
| 配置文件权限   | `640 root:redis` |
| ACL 文件权限 | `640 root:redis` |


### 15.5 持久化检查


| 检查项                      | 期望         |
| ------------------------ | ---------- |
| `rdb_last_bgsave_status` | `ok`       |
| `aof_last_write_status`  | `ok`       |
| 数据目录空间                   | 充足         |
| 数据目录权限                   | `redis` 可写 |
| 重启后数据                    | 符合预期       |


---

## 16. 运维检查命令

以下命令建议纳入日常巡检或监控。

### 16.1 服务状态

```bash
systemctl status redis --no-pager
journalctl -u redis -n 100 --no-pager
```

### 16.2 Redis 基础状态

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO server
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO clients
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO memory
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO stats
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO persistence
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor SLOWLOG LEN
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor SLOWLOG GET 10
unset REDISCLI_AUTH
```

### 16.3 重点监控指标


| 指标                                  | 风险说明        |
| ----------------------------------- | ----------- |
| `used_memory`                       | Redis 已使用内存 |
| `used_memory_rss`                   | 进程实际占用内存    |
| `mem_fragmentation_ratio`           | 内存碎片率       |
| `connected_clients`                 | 当前客户端连接数    |
| `blocked_clients`                   | 阻塞客户端数量     |
| `rejected_connections`              | 被拒绝连接数      |
| `evicted_keys`                      | 被淘汰 key 数量  |
| `expired_keys`                      | 过期 key 数量   |
| `keyspace_hits` / `keyspace_misses` | 缓存命中率       |
| `rdb_last_bgsave_status`            | RDB 保存状态    |
| `aof_last_write_status`             | AOF 写入状态    |
| `latest_fork_usec`                  | 最近 fork 耗时  |


---

## 17. 停止、重启与回滚

### 17.1 停止 Redis

优先使用 systemd：

```bash
systemctl stop redis
systemctl status redis --no-pager
```

不建议在生产规范中直接使用明文密码执行 `shutdown`。

### 17.2 重启 Redis

单体 Redis 存在单点风险，生产重启前必须确认业务可接受影响窗口。

```bash
systemctl restart redis
systemctl status redis --no-pager
```

### 17.3 配置变更前

备份配置文件：

```bash
cp /data/module/redis7.2/conf/redis.conf /data/module/redis7.2/conf/redis.conf.bak.$(date +%F-%H%M%S)
cp /data/module/redis7.2/conf/users.acl /data/module/redis7.2/conf/users.acl.bak.$(date +%F-%H%M%S)
```

配置变更仍需在测试环境验证。Redis 没有等价于 Nginx `nginx -t` 的完整配置检查命令，不要把 `--test-memory` 误当成配置语法检查。

### 17.4 回滚配置

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

### 17.5 版本回滚

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

## 18. 常见问题

### Q1：编译报错 jemalloc 相关怎么办？

优先排查编译环境、源码包完整性和系统依赖。若短时间无法处理，可使用 libc 编译作为临时方案：

```bash
cd /data/module/redis7.2.14/src/redis-7.2.14
make distclean
make MALLOC=libc -j$(nproc)
make install PREFIX=/data/module/redis7.2.14
ln -sfn /data/module/redis7.2.14 /data/module/redis7.2
```

生产环境推荐使用默认 jemalloc。使用 libc 前应在测试环境完成性能与内存碎片评估。

### Q2：如何确认当前 Redis 版本？

```bash
/data/module/redis7.2/bin/redis-server --version
/data/module/redis7.2/bin/redis-cli --version
```

Redis 运行后也可以查看：

```bash
export REDISCLI_AUTH='wyz123!@#'
/data/module/redis7.2/bin/redis-cli -h 10.10.10.10 -p 16379 --user monitor INFO server | grep redis_version
unset REDISCLI_AUTH
```

### Q3：为什么不在生产服务器上直接 wget？

企业内网部署应使用受控制品来源，避免生产服务器直接访问公网。安装包应先进入企业制品库或经过审批的交付目录，再上传到目标服务器安装。

### Q4：为什么不用 `requirepass`？

Redis 6+ 已支持 ACL。生产环境使用 ACL 可以区分应用、监控、运维等不同账号与权限，比单一 `requirepass` 更适合审计和权限收敛。

### Q5：为什么不使用 `daemonize yes`？

生产环境由 systemd 托管 Redis 时，应使用：

```conf
daemonize no
supervised systemd
```

这样 systemd 可以正确感知 Redis 主进程状态、统一管理重启策略、资源限制和日志。