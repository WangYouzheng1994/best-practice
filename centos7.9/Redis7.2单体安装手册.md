# Redis 7.2.14 单体安装手册

> author: 王有政  
> encoding: UTF-8

## 概述

本文档面向 **CentOS 7.9**，用于在企业内网环境手动安装 **Redis 7.2.14** 单体实例。

本文只保留源码包上传、解压、编译、安装和基础启动验证流程，不包含容器部署方式。

约定路径：

| 类型 | 路径 |
|---|---|
| 安装根目录 | `/data/module` |
| Redis 安装目录 | `/data/module/redis7.2.14` |
| Redis 源码包 | `/data/module/redis-7.2.14.tar.gz` |
| Redis 解压目录 | `/data/module/redis7.2.14/redis-7.2.14` |
| 配置文件 | `/data/module/redis7.2.14/conf/redis.conf` |
| 本地实例配置 | `/data/module/redis7.2.14/conf/redis-local.conf` |
| 数据目录 | `/data/module/redis7.2.14/data` |
| 日志目录 | `/data/module/redis7.2.14/logs` |

## 一、安装前准备

### 1. 检查安装包

企业内网服务器通常不能直接访问外网，不要在生产服务器上使用 `wget` 或 `git clone` 从 GitHub 拉取源码。

应提前通过企业制品库、堡垒机文件上传、SFTP、SCP、`rz` 等方式，将 `redis-7.2.14.tar.gz` 上传到 `/data/module`。

```bash
cd /data/module
ls -lh redis-7.2.14.tar.gz
```

如需使用 `rz` 上传，先确认服务器已有 `lrzsz`：

```bash
rpm -q lrzsz || yum install -y lrzsz
cd /data/module
rz -be
```

> 上传完成后，建议按企业交付清单校验 `sha256sum redis-7.2.14.tar.gz`。校验值不一致时不要继续安装。

### 2. 安装编译依赖

```bash
yum install -y gcc make
```

> `tcl` 只用于执行 `make test`，生产安装 Redis 不强制要求安装。

### 3. 系统初始化要求

安装 Redis 前，服务器必须已完成 `CentOS7.9生产环境初始化手册.md` 中的系统初始化配置，至少包括：

- Swap 与内存管理；
- Transparent Huge Pages 关闭；
- 内核参数优化；
- 文件句柄数限制；
- 数据盘挂载参数与 I/O 调度器配置。

Redis 安装手册只维护 Redis 软件安装和 Redis 实例配置，不重复维护操作系统级参数。

### 4. 生成生产环境强密码

Redis 安装前先生成强密码，并用生成结果替换后文所有 `ChangeMe_StrongPassword`。

#### 方案 A：生成包含特殊字符的复杂密码（推荐）

如果企业密码规范要求必须包含大小写字母、数字和特殊字符，推荐使用以下方式生成：

```bash
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:,.~!#$^*-' < /dev/urandom | head -c 48; echo
```

该命令会生成类似 `P@ssw0rd` 风格但真正随机的复杂密码，满足大小写字母、数字、特殊字符等常见密码复杂度要求。字符集刻意避开了空格、单引号、双引号、反斜杠、反引号、分号、管道符等容易影响配置文件或命令行解析的字符。可通过调整 `head -c 48` 控制密码长度，例如改为 `head -c 32` 生成 32 位密码。

#### 方案 B：生成十六进制随机密码

```bash
openssl rand -hex 32
```

该方式生成 64 位十六进制字符，随机强度高，且不包含空格、引号、反斜杠、美元符号、感叹号等容易被 Shell 误解析的特殊字符，适合写入 Redis 配置文件和运维命令。但它只包含 `0-9a-f`，不满足部分企业对“必须包含大小写字母、数字、特殊字符”的密码复杂度要求。

#### 方案 C：没有 OpenSSL 时生成普通随机密码

```bash
tr -dc 'A-Za-z0-9_@%+=:,.~-' < /dev/urandom | head -c 48; echo
```

密码要求：不少于 32 字符，必须随机生成。不要使用人工编造的 `P@ssw0rd`、`Redis@123`、`Admin@123456`、姓名、项目名、公司名、手机号、生日、常见单词等可猜测内容。

推荐优先使用 **方案 A**。如果现场更重视命令行和配置文件转义安全，且企业密码策略不强制要求特殊字符，可使用 **方案 B**。真实密码不要写入文档、提交到 Git 仓库或发送到群聊，应保存到企业密码管理系统或受控密钥管理平台。

## 二、解压、编译、安装

### 1. 解压源码包

```bash
cd /data/module
mkdir -p /data/module/redis7.2.14
tar -zxvf redis-7.2.14.tar.gz -C /data/module/redis7.2.14
cd /data/module/redis7.2.14/redis-7.2.14
```

### 2. 编译

```bash
make -j$(nproc)
```

> Redis 默认使用 jemalloc，生产环境推荐保留默认行为。只有在 jemalloc 编译失败且短时间无法处理时，才考虑执行 `make distclean` 后使用 `make MALLOC=libc -j$(nproc)` 重新编译。

### 3. 安装到 `/data/module/redis7.2.14`

```bash
mkdir -p /data/module/redis7.2.14
make install PREFIX=/data/module/redis7.2.14
```

安装完成后确认二进制文件：

```bash
ls -lh /data/module/redis7.2.14/bin
/data/module/redis7.2.14/bin/redis-server --version
```

## 三、配置 Redis

### 1. 创建运行目录

```bash
mkdir -p /data/module/redis7.2.14/{conf,data,logs}
```

### 2. 复制配置文件并写入本地配置

先从解压后的源码目录复制官方配置文件，保留 Redis 原始配置模板和注释说明：

```bash
cp /data/module/redis7.2.14/redis-7.2.14/redis.conf /data/module/redis7.2.14/conf/redis.conf
cp /data/module/redis7.2.14/conf/redis.conf /data/module/redis7.2.14/conf/redis.conf.bak
```

再写入本地实例配置，并在官方配置文件末尾引入该配置：

```bash
cat > /data/module/redis7.2.14/conf/redis-local.conf << 'EOF'
bind 0.0.0.0
port 6379
protected-mode yes
requirepass ChangeMe_StrongPassword

daemonize yes
supervised no
pidfile /data/module/redis7.2.14/redis_6379.pid

loglevel notice
logfile /data/module/redis7.2.14/logs/redis.log

dir /data/module/redis7.2.14/data

databases 16
maxclients 10000
timeout 0
tcp-keepalive 300

save ""
save 900 1
save 300 10
save 60 10000
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes

# 生产建议将 maxmemory 设置为物理内存的 60%～70%，最高不建议超过 75%
# 示例：8GB 内存机器可设置 5gb～6gb，需给系统、页缓存、RDB/AOF rewrite、fork 预留内存
maxmemory 6gb
maxmemory-policy allkeys-lru

slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 100
EOF

cat >> /data/module/redis7.2.14/conf/redis.conf << 'EOF'

include /data/module/redis7.2.14/conf/redis-local.conf
EOF

chmod 640 /data/module/redis7.2.14/conf/redis.conf
chmod 640 /data/module/redis7.2.14/conf/redis-local.conf
```

> 这里保留 `redis.conf` 作为官方基础模板，业务侧只维护 `redis-local.conf`。其中 `save ""` 用于先清空官方模板里的默认 RDB 保存策略，再按本手册重新定义保存策略。

## 四、启动与验证

### 1. 启动 Redis

```bash
/data/module/redis7.2.14/bin/redis-server /data/module/redis7.2.14/conf/redis.conf
```

### 2. 验证进程和端口

```bash
ps -ef | grep redis-server | grep -v grep
ss -lntp | grep 6379
```

### 3. 验证客户端连接

```bash
/data/module/redis7.2.14/bin/redis-cli -a 'ChangeMe_StrongPassword' ping
```

期望输出：

```text
PONG
```

### 4. 查看日志

```bash
tail -n 100 /data/module/redis7.2.14/logs/redis.log
```

## 五、停止与重启

### 1. 停止 Redis

```bash
/data/module/redis7.2.14/bin/redis-cli -a 'ChangeMe_StrongPassword' shutdown
```

### 2. 重启 Redis

```bash
/data/module/redis7.2.14/bin/redis-server /data/module/redis7.2.14/conf/redis.conf
```

## 六、常见问题

### Q1: 编译报错 jemalloc 相关怎么办？

```bash
cd /data/module/redis7.2.14/redis-7.2.14
make distclean
make MALLOC=libc -j$(nproc)
make install PREFIX=/data/module/redis7.2.14
```

### Q2: 如何确认当前 Redis 版本？

```bash
/data/module/redis7.2.14/bin/redis-server --version
/data/module/redis7.2.14/bin/redis-cli -a 'ChangeMe_StrongPassword' info server | grep redis_version
```

### Q3: 为什么不在生产服务器上直接 wget？

企业内网部署应使用受控制品来源，避免生产服务器直接访问公网。安装包应先进入企业制品库或经过审批的交付目录，再上传到目标服务器安装。
