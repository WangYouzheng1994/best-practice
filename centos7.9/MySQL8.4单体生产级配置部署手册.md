# MySQL 8.4 单体生产级配置部署手册

> **Author:** 王有政  
> **日期:** 2026-06-15  
> **适用系统:** CentOS 7.9 (x86_64)  
> **MySQL 版本:** MySQL Community Server 8.4 LTS  
> **文档定位:** 单体 MySQL 8.4 生产级部署与参数配置。主从复制、MGR、备份平台、跨机房容灾请拆分为独立文档维护。

---

## 目录

1. [适用范围与部署原则](#1-适用范围与部署原则)
2. [部署前检查](#2-部署前检查)
3. [安装方案选择](#3-安装方案选择)
4. [系统依赖与环境准备](#4-系统依赖与环境准备)
5. [安装 MySQL 8.4](#5-安装-mysql-84)
6. [目录、用户与权限](#6-目录用户与权限)
7. [MySQL 生产参数配置](#7-mysql-生产参数配置)
8. [初始化数据库](#8-初始化数据库)
9. [systemd 服务配置](#9-systemd-服务配置)
10. [启动与安全加固](#10-启动与安全加固)
11. [网络边界与访问控制](#11-网络边界与访问控制)
12. [开发环境 SQL 快速还原](#12-开发环境-sql-快速还原)
13. [生产验证清单](#13-生产验证清单)
14. [运维检查命令](#14-运维检查命令)
15. [变更与回滚建议](#15-变更与回滚建议)
16. [常见问题](#16-常见问题)

---

## 1. 适用范围与部署原则

本文用于部署 **单体 MySQL 8.4 LTS**，适用于以下场景：

- 中小规模业务的单机数据库；
- 开发、测试、预生产环境；
- 生产环境中暂不具备高可用条件，但需要规范化部署的数据库；
- 后续作为主从复制、MGR 或备份恢复体系的基础节点模板。

本文不覆盖以下内容：

- 主从复制；
- 半同步复制；
- MySQL Group Replication；
- InnoDB Cluster；
- 跨机房容灾；
- 备份平台建设。

生产级单体 MySQL 必须遵循以下原则：

1. MySQL 进程不得使用 `root` 用户运行；
2. 数据目录、日志目录、临时目录必须与程序二进制目录分开管理，统一收敛到 MySQL 实例目录下；
3. 数据盘建议独立挂载到 `/data`，禁止与系统盘混用承载高负载生产库；
4. MySQL 端口不得直接暴露到公网，必须由外层防火墙、安全组、VPN、堡垒机或统一网络策略保护；
5. 禁止使用 MySQL 默认端口 `3306`，必须遵循《服务器端口统一规划规范.md》，默认使用 `13300 ~ 13399` 端口段，单体标准端口为 `13306`；
6. 远程 `root` 登录必须配合防火墙、安全组、强密码和审计措施使用；
7. 应使用独立业务账号，并按最小权限授权；
8. 必须开启 binlog，为误操作恢复、审计和后续复制预留能力；
9. 必须配置慢查询日志；
10. 必须设置合理的连接数、内存、redo、临时表和日志保留参数；
11. 必须由 systemd 托管并设置开机自启；
12. 必须限制访问来源 IP；
13. 必须完成启动验证、安全验证、参数验证和基础读写验证。

> 重要说明：本文仅面向 CentOS 7.9 存量环境，部署步骤、命令和配置均按 CentOS 7.9 编写。

---

## 2. 部署前检查

### 2.1 服务器资源建议


| 资源   | 最低要求       | 生产建议                |
| ---- | ---------- | ------------------- |
| CPU  | 4 核        | 8 核及以上              |
| 内存   | 8 GB       | 16 GB 及以上           |
| 系统盘  | 50 GB SSD  | 100 GB SSD          |
| 数据盘  | 100 GB SSD | 独立 SSD/NVMe，按业务容量规划 |
| 文件系统 | xfs/ext4   | xfs                 |
| 网络   | 千兆内网       | 万兆内网                |
| 时间同步 | 必须启用       | chrony              |


检查基础信息：

```bash
cat /etc/redhat-release
uname -r
hostnamectl
lscpu
free -h
lsblk
blkid
df -hT
ip addr
systemctl status chronyd --no-pager
```

### 2.2 glibc 版本检查

CentOS 7.9 默认 glibc 为 2.17。安装 MySQL 二进制包时必须选择兼容 glibc 版本的软件包；如果使用官方 Yum 仓库安装，通常由仓库自动处理依赖。

```bash
ldd --version | head -1
rpm -qa | grep '^glibc'
```

### 2.3 确认是否存在 MariaDB 或旧 MySQL

CentOS 7 默认可能安装 MariaDB 库或服务。安装前必须确认是否存在历史数据，禁止直接删除未知实例。

```bash
rpm -qa | grep -Ei 'mariadb|mysql|percona'
systemctl list-unit-files | grep -Ei 'mariadb|mysql|mysqld'
ps -ef | grep -E '[m]ysqld|[m]ariadbd'
ss -lntp | grep ':13306'
```

如果存在旧实例，先确认以下内容：

- 是否仍在承载业务；
- 数据目录位置；
- 是否已有备份；
- 是否需要迁移或保留配置；
- 是否允许停机。

---

## 3. 安装方案选择

MySQL 8.4 在 Linux 上常见安装方式有三种。


| 方案                   | 说明                                | 优点                                 | 缺点                               | 适用场景              |
| -------------------- | --------------------------------- | ---------------------------------- | -------------------------------- | ----------------- |
| 方案 A：Generic 二进制包安装  | 使用官方 Linux Generic `tar.xz` 包解压部署 | 不依赖生产服务器外网；安装路径统一；版本切换和回滚直观；适合离线交付 | 需要团队自行维护 systemd、依赖、升级、卸载和资产登记规范 | 本文推荐，适合团队标准化部署    |
| 方案 B：内网 Yum/RPM 仓库安装 | 将官方 RPM 同步到公司内网仓库                 | 包管理、审计、升级能力最好                      | 需要建设和维护内网仓库                      | 大规模成熟运维团队         |
| 方案 C：RPM Bundle 离线安装 | 下载官方 RPM Bundle 后离线安装             | 无需生产服务器外网；仍保留 RPM 包管理能力            | 依赖顺序和包版本需要维护；路径灵活性较弱             | 无内网仓库但希望保留 RPM 管理 |


**推荐：** 本文按 **方案 A：Generic 二进制包安装** 编写主流程。该方案适合生产环境无外网、团队希望统一安装目录、统一 systemd 模板、统一升级回滚方式的场景。

采用 Generic 二进制方式时，团队必须统一以下规范：

1. 软件包必须来自官方可信来源或公司制品库；
2. 软件包必须进行 SHA256 校验；
3. 版本目录必须带完整版本号；
4. 运行入口必须通过稳定软链接暴露；
5. systemd 服务必须显式指定 `basedir`、`datadir` 和配置文件；
6. 升级必须通过新版本目录和软链接切换完成；
7. 回滚必须保留旧版本目录和旧配置备份；
8. 资产台账必须记录 MySQL 版本、包名、校验值、部署路径和部署时间。

本文采用以下标准目录：

```text
/data/                                             # 示例数据盘挂载点，实际以现场挂载点为准
/data/module/                                      # 软件安装目录、当前版本软链接与 MySQL 实例目录
/data/module/mysql-8.4.9-linux-glibc2.17-x86_64/  # MySQL 具体版本目录
/data/module/mysql                                # MySQL 当前版本软链接与实例根目录
/data/module/mysql/data/                           # MySQL 数据目录
/data/module/mysql/binlog/                         # MySQL binlog 目录
/data/module/mysql/relaylog/                       # MySQL relaylog 目录，后续复制预留
/data/module/mysql/undo/                           # MySQL undo tablespace 目录
/data/module/mysql/tmp/                            # MySQL 临时目录
/data/module/mysql/logs/                           # MySQL 实例日志目录
/data/module/mysql/backup                          # MySQL 本机临时备份目录
/data/module/mysql/run/                            # MySQL socket 与 pid 运行时目录
/data/module/mysql/conf/my.cnf                     # 主配置文件
/etc/systemd/system/mysqld.service                # systemd 服务文件
```

> 说明：`/data` 只是本文示例的数据盘挂载点，不代表所有服务器都必须叫 `/data`。如果现场数据盘挂载到 `/data1`、`/mnt/data`、`/u01` 等路径，应按实际挂载点统一替换。本文将 MySQL 软件、配置、数据、日志、备份和运行时目录统一收敛到 `/data/module/mysql` 实例根目录下，便于单实例部署、巡检、迁移和备份。生产环境推荐保留带版本号的真实目录，并通过 `/data/module/mysql` 软链接暴露当前版本。不建议将解压目录直接改名为 `mysql`，也不建议解压时使用 `--strip-components` 直接覆盖到固定目录。这样可以保留版本信息，便于升级、回滚、资产登记和故障排查；systemd、环境变量和脚本只引用 `/data/module/mysql`，升级时通常只需要切换软链接。

---

## 4. 系统依赖与环境准备

### 4.1 系统初始化要求

安装 MySQL 前，服务器必须已完成 `CentOS7.9生产环境初始化手册.md` 中的系统初始化配置，至少包括：

- 数据盘独立挂载到 `/data`，并完成挂载参数优化；
- Swap 与内存管理策略；
- Transparent Huge Pages 关闭；
- 内核参数优化；
- 文件句柄数限制；
- 磁盘 I/O 调度器配置；
- 时间同步和基础安全加固。

MySQL 部署手册只维护 MySQL 软件安装、实例目录、MySQL 参数和服务托管配置，不重复维护操作系统级参数。

### 4.2 安装 MySQL 运行依赖

Generic 二进制包不负责安装系统依赖，需要手动确认依赖齐全。

```bash
yum install -y \
  lsof net-tools tar xz \
  libaio numactl-libs ncurses-compat-libs \
  openssl openssl-libs \
  perl perl-Data-Dumper perl-JSON
```

如果 `ncurses-compat-libs` 在当前 Yum 源不可用，可先执行：

```bash
yum provides '*/libncurses.so.5'
yum provides '*/libtinfo.so.5'
```

### 4.3 创建数据目录规划

本文采用以下目录规划：


| 类型          | 路径                               | 说明                             |
| ----------- | -------------------------------- | ------------------------------ |
| 配置文件        | `/data/module/mysql/conf/my.cnf` | MySQL 主配置文件                    |
| 数据目录        | `/data/module/mysql/data`        | InnoDB 数据文件与系统表空间              |
| binlog 目录   | `/data/module/mysql/binlog`      | 二进制日志                          |
| relaylog 目录 | `/data/module/mysql/relaylog`    | 后续复制预留                         |
| undo 目录     | `/data/module/mysql/undo`        | undo tablespace                |
| 临时目录        | `/data/module/mysql/tmp`         | MySQL 临时文件                     |
| 日志目录        | `/data/module/mysql/logs`        | error log、slow log、general log |
| socket 目录   | `/data/module/mysql/run`         | socket 与 pid 文件                |
| 备份目录        | `/data/module/mysql/backup`      | 本机临时备份落盘目录                     |


创建目录：

```bash
mkdir -p /data/module/mysql/{conf,data,binlog,relaylog,undo,tmp,logs,backup,run}
```

`/data/module/mysql/run` 为运行时目录，本文统一放在 MySQL 实例根目录下，便于单应用维护和巡检。

### 4.4 确认数据盘挂载

生产环境建议将 `/data` 挂载到独立数据盘，并使用 `xfs` 文件系统。挂载和 I/O 调度器配置应在 `CentOS7.9生产环境初始化手册.md` 中完成，本文只做确认。

```bash
df -hT /data
findmnt /data
```

> 注意：如果 `/data` 未挂载到独立数据盘，应先完成系统初始化和数据盘挂载，再继续部署 MySQL。

---

## 5. 安装 MySQL 8.4

### 5.1 获取 Generic 二进制安装包

在有外网的办公机、跳板机或制品构建机上下载 MySQL 8.4 Linux Generic 二进制包，然后上传到公司制品库或目标服务器。

CentOS 7.9 默认 glibc 为 2.17，应选择兼容 glibc 2.17 的 Linux Generic 包。若官方后续不再提供 glibc 2.17 包，应停止在 CentOS 7.9 上新部署 MySQL 8.4，改用 EL8/EL9 系统，不建议在生产服务器上手工升级 glibc。

目标服务器标准存放目录：

```bash
mkdir -p /data/module
cd /data/module
```

如需使用 `rz` 上传：

```bash
rz -be
```

上传后的文件示例：

```text
/data/module/mysql-8.4.9-linux-glibc2.17-x86_64.tar.xz
/data/module/mysql-8.4.9-linux-glibc2.17-x86_64.tar.xz.sha256
```

> 说明：本文按团队当前使用的 `8.4.9` 编写。生产环境不要使用来源不明的第三方重打包版本。

### 5.2 校验安装包

如果有 `.sha256` 文件，使用以下命令校验：

```bash
cd /data/module
sha256sum -c mysql-8.4.9-linux-glibc2.17-x86_64.tar.xz.sha256
```

如果制品库只提供 SHA256 字符串，则使用以下方式比对：

```bash
sha256sum mysql-8.4.9-linux-glibc2.17-x86_64.tar.xz
```

校验通过后再继续安装。校验值应登记到部署记录或资产台账中。

### 5.3 解压到标准目录

解压安装包：

```bash
cd /data/module
tar -xJvf mysql-8.4.9-linux-glibc2.17-x86_64.tar.xz -C /data/module/
```

创建稳定软链接：

```bash
cd /data/module
ln -sfn mysql-8.4.9-linux-glibc2.17-x86_64 mysql
```

确认目录：

```bash
ls -ld /data/module/mysql-8.4.9-linux-glibc2.17-x86_64 /data/module/mysql
ls -l /data/module/mysql/bin/mysql /data/module/mysql/bin/mysqld
```

### 5.4 配置环境变量

团队统一将自定义环境变量维护在 `/etc/profile.d/myenv.sh`，不要为 MySQL 单独创建 `/etc/profile.d/mysql.sh`。

编辑统一环境变量文件：

```bash
vim /etc/profile.d/myenv.sh
```

追加以下内容：

```bash
# MYSQL_HOME：MySQL 当前版本软链接目录，统一指向 /data/module/mysql，升级时只切换软链接
export MYSQL_HOME=/data/module/mysql

# PATH：将 MySQL 客户端和服务端命令加入命令搜索路径，便于直接执行 mysql、mysqld、mysqladmin 等命令
export PATH=$MYSQL_HOME/bin:$PATH
```

保存后加载生效：

```bash
chmod 644 /etc/profile.d/myenv.sh
source /etc/profile.d/myenv.sh
```

验证版本：

```bash
mysql --version
mysqld --version
which mysql
which mysqld
```

期望 `which mysql` 和 `which mysqld` 指向 `/data/module/mysql/bin/`。

### 5.5 检查动态库依赖

```bash
ldd /data/module/mysql/bin/mysqld | grep 'not found' || true
ldd /data/module/mysql/bin/mysql | grep 'not found' || true
```

如果存在 `not found`，必须补齐依赖后再初始化。

### 5.6 清理默认 MariaDB 干扰

如果系统中存在 MariaDB 包，需要谨慎处理。仅在确认没有历史业务后执行卸载。

```bash
rpm -qa | grep -Ei '^mariadb|^mysql'
```

确认可删除后：

```bash
systemctl stop mariadb 2>/dev/null || true
yum remove -y mariadb-server mariadb
```

> 注意：不建议盲目卸载 `mariadb-libs`，它可能被系统或其他软件依赖。Generic 二进制部署不要求卸载 `mariadb-libs`。

---

## 6. 目录、用户与权限

Generic 二进制安装不会自动创建 `mysql` 用户，需要手动创建：

```bash
id mysql || useradd -r -s /sbin/nologin mysql
```

设置安装目录和实例目录权限：

```bash
chown -R root:root /data/module/mysql-8.4.9-linux-glibc2.17-x86_64
chown -h root:root /data/module/mysql
chown -R mysql:mysql /data/module/mysql/{data,binlog,relaylog,undo,tmp,logs,backup,run}
chmod 755 /data/module/mysql-8.4.9-linux-glibc2.17-x86_64
chmod 750 /data/module/mysql/{data,binlog,relaylog,undo,tmp,logs,backup,run}
```

`/data/module/mysql/run` 统一作为 MySQL socket 与 pid 目录，由部署步骤创建并授权给 `mysql` 用户。

确认权限：

```bash
ls -ld /data/module/mysql /data/module/mysql-8.4.9-linux-glibc2.17-x86_64
ls -ld /data/module/mysql/{conf,data,binlog,relaylog,undo,tmp,logs,backup,run}
```

---

## 7. MySQL 生产参数配置

### 7.1 写入 `/data/module/mysql/conf/my.cnf`

写入配置前先备份原文件；如果已存在 `my.cnf`，则生成一份带时间戳的备份：

```bash
[ -f /data/module/mysql/conf/my.cnf ] && cp /data/module/mysql/conf/my.cnf /data/module/mysql/conf/my.cnf.bak.$(date +%F-%H%M%S)
```

创建生产单体配置：

```bash
cat > /data/module/mysql/conf/my.cnf << 'EOF'
[client]
port = 13306
socket = /data/module/mysql/run/mysql.sock
default-character-set = utf8mb4

[mysql]
prompt = "\\u@\\h [\\d]> "
default-character-set = utf8mb4
show-warnings

[mysqld]
# basic
user = mysql
port = 13306
server_id = 1001
basedir = /data/module/mysql
datadir = /data/module/mysql/data
socket = /data/module/mysql/run/mysql.sock
pid-file = /data/module/mysql/run/mysqld.pid
tmpdir = /data/module/mysql/tmp

# network
bind-address = 0.0.0.0
mysqlx = 0
max_connections = 800
max_connect_errors = 100000
back_log = 1024
connect_timeout = 10
wait_timeout = 600
interactive_timeout = 600
net_read_timeout = 60
net_write_timeout = 60
max_allowed_packet = 64M

# charset and sql mode
character_set_server = utf8mb4
collation_server = utf8mb4_0900_ai_ci
init_connect = 'SET NAMES utf8mb4'
default_time_zone = '+08:00'
sql_mode = ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION

# table cache
open_files_limit = 65535
table_open_cache = 4000
table_definition_cache = 2000
table_open_cache_instances = 16

# memory
thread_cache_size = 128
tmp_table_size = 64M
max_heap_table_size = 64M
sort_buffer_size = 4M
join_buffer_size = 4M
read_buffer_size = 2M
read_rnd_buffer_size = 4M

# innodb core
innodb_buffer_pool_size = 6G
innodb_buffer_pool_instances = 6
innodb_data_file_path = ibdata1:1G:autoextend
innodb_undo_directory = /data/module/mysql/undo
innodb_file_per_table = 1
innodb_open_files = 4000
innodb_flush_method = O_DIRECT
innodb_flush_log_at_trx_commit = 1
innodb_doublewrite = 1
innodb_page_cleaners = 4
innodb_io_capacity = 1000
innodb_io_capacity_max = 4000
innodb_read_io_threads = 4
innodb_write_io_threads = 4
innodb_thread_concurrency = 0
innodb_lock_wait_timeout = 50
innodb_print_all_deadlocks = 1

# redo log
innodb_redo_log_capacity = 4G

# binlog
log_bin = /data/module/mysql/binlog/mysql-bin
binlog_format = ROW
binlog_row_image = FULL
binlog_expire_logs_seconds = 604800
sync_binlog = 1
max_binlog_size = 1G
log_bin_trust_function_creators = 0

# relay log reserved for future replication
relay_log = /data/module/mysql/relaylog/mysql-relay-bin
relay_log_recovery = 1

# logs
log_error = /data/module/mysql/logs/error.log
slow_query_log = 1
slow_query_log_file = /data/module/mysql/logs/slow.log
long_query_time = 1
log_queries_not_using_indexes = 0
log_slow_admin_statements = 1
log_slow_replica_statements = 1
general_log = 0
general_log_file = /data/module/mysql/logs/general.log

# performance schema
performance_schema = ON

# security
local_infile = 0
skip_name_resolve = 1
symbolic-links = 0
secure_file_priv = /data/module/mysql/tmp

# crash safe and compatibility
explicit_defaults_for_timestamp = 1
lower_case_table_names = 1

[mysqldump]
quick
single-transaction
max_allowed_packet = 64M

[myisamchk]
key_buffer_size = 64M
sort_buffer_size = 64M
read_buffer = 2M
write_buffer = 2M
EOF
```

设置配置文件权限：

```bash
chown root:root /data/module/mysql/conf/my.cnf
chmod 644 /data/module/mysql/conf/my.cnf
```

> 注意：`bind-address = 0.0.0.0` 表示监听本机所有 IPv4 地址，便于开发、运维工具和内网服务连接。该配置要求服务器外层已有防火墙、安全组、ACL、VPN、堡垒机或统一网络边界保护，禁止将 MySQL 端口直接暴露到公网。`port = 13306` 为本文示例端口，生产环境不得使用 MySQL 默认端口 `3306`。

### 7.2 端口规划要求

MySQL 生产实例禁止使用默认端口 `3306`。端口规划必须遵循《服务器端口统一规划规范.md》，MySQL 统一使用 `13300 ~ 13399` 端口段，单体标准端口为 `13306`。实际部署时应由团队在资产台账自行分配，确保端口不冲突、可审计、可回溯。

端口选择建议：


| 方案   | 示例              | 说明                                               |
| ---- | --------------- | ------------------------------------------------ |
| 方案 A | `13306`         | 单实例标准端口，便于识别且避开默认端口                              |
| 方案 B | `13306 ~ 13399` | 多实例场景在 MySQL 端口段内连续规划，例如 `13306`、`13307`、`13308` |
| 方案 C | CMDB 分配         | 由资产平台统一分配和登记，适合规模化环境                             |


**推荐：** 单体标准化部署选择方案 A；如果同一主机存在多实例，选择方案 B；规模化环境优先选择方案 C。无论采用哪种方案，都不得回退到 `3306`。

端口变更时，必须同步修改以下位置：

- `/data/module/mysql/conf/my.cnf` 中 `[client]` 和 `[mysqld]` 的 `port`；
- 启动、验证和巡检命令中的端口检查；
- 外层防火墙、安全组、负载均衡或堡垒机访问策略；
- CMDB、资产台账、监控系统和应用连接串。

### 7.3 核心参数决策

#### 7.3.1 `innodb_buffer_pool_size`


| 方案   | 建议值          | 适用场景                  |
| ---- | ------------ | --------------------- |
| 方案 A | 物理内存 50%     | 数据库与应用混部，或内存小于 16 GB  |
| 方案 B | 物理内存 60%~70% | 单体 MySQL 独占服务器，常规生产推荐 |
| 方案 C | 物理内存 75%     | 大内存且实例独占，连接数和临时表可控    |


**推荐：** 选择方案 B。示例：16 GB 内存设置 `10G`，32 GB 内存设置 `20G`，64 GB 内存设置 `40G`。

#### 7.3.2 `max_connections`


| 方案   | 建议值      | 适用场景             |
| ---- | -------- | ---------------- |
| 方案 A | 300~500  | 小型业务，连接池规范       |
| 方案 B | 800~1500 | 常规生产单体           |
| 方案 C | 2000+    | 高并发短连接场景，但必须评估内存 |


**推荐：** 选择方案 B，并优先在应用侧使用连接池控制并发。不要把 `max_connections` 当作解决慢 SQL 的手段。

#### 7.3.3 `innodb_flush_log_at_trx_commit` 与 `sync_binlog`


| 方案   | 参数组合                                                  | 数据安全 | 性能  | 适用场景            |
| ---- | ----------------------------------------------------- | ---- | --- | --------------- |
| 方案 A | `innodb_flush_log_at_trx_commit=1`，`sync_binlog=1`    | 最高   | 较低  | 金融、订单、核心交易      |
| 方案 B | `innodb_flush_log_at_trx_commit=1`，`sync_binlog=100`  | 高    | 中   | 可接受少量 binlog 风险 |
| 方案 C | `innodb_flush_log_at_trx_commit=2`，`sync_binlog=1000` | 中    | 高   | 可重建数据、日志型业务     |


**推荐：** 核心生产库选择方案 A。非核心但写入压力较大时，可经过压测和业务确认后选择方案 B。

#### 7.3.4 `binlog_expire_logs_seconds`


| 方案   | 建议值     | 说明      |
| ---- | ------- | ------- |
| 方案 A | 259200  | 保留 3 天  |
| 方案 B | 604800  | 保留 7 天  |
| 方案 C | 1209600 | 保留 14 天 |


**推荐：** 选择方案 B。前提是磁盘容量充足，并配合备份策略。

#### 7.3.5 `lower_case_table_names`

该参数必须在初始化前确定，初始化后不应修改。


| 方案   | 建议值 | 说明                    |
| ---- | --- | --------------------- |
| 方案 A | `0` | CentOS 7.9 默认，表名大小写敏感 |
| 方案 B | `1` | 表名以小写存储，查询不区分大小写      |


**推荐：** 新系统选择方案 A，并在研发规范中统一表名全部小写。如果需要兼容历史迁移库，可在初始化前评估方案 B。

### 7.4 按服务器内存调整示例


| 物理内存   | `innodb_buffer_pool_size` | `innodb_buffer_pool_instances` | `max_connections` |
| ------ | ------------------------- | ------------------------------ | ----------------- |
| 8 GB   | 4G                        | 4                              | 300               |
| 16 GB  | 10G                       | 8                              | 800               |
| 32 GB  | 20G                       | 8                              | 1000              |
| 64 GB  | 40G                       | 8                              | 1500              |
| 128 GB | 80G                       | 8                              | 2000              |


修改配置后检查语法：

```bash
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --validate-config
```

---

## 8. 初始化数据库

### 8.1 初始化前确认

初始化会创建系统库。必须确认数据目录为空。

> 注意：`lower_case_table_names = 1` 必须在初始化数据目录前确定。初始化完成后不要直接修改该参数，避免 MySQL 数据字典和表名大小写规则不一致。

```bash
ls -la /data/module/mysql/data
ls -la /data/module/mysql/undo
```

如果目录已有内容，不要直接删除，先确认是否是历史实例。

### 8.2 执行初始化

推荐使用安全初始化方式，生成临时 root 密码：

```bash
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --initialize --user=mysql
```

查看临时密码：

```bash
grep 'temporary password' /data/module/mysql/logs/error.log
```

如果只是测试环境，也可以使用无密码初始化，但生产环境不推荐：

```bash
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --initialize-insecure --user=mysql
```

---

## 9. systemd 服务配置

Generic 二进制安装不会自带 systemd 服务文件，需要手动创建。

```bash
cat > /etc/systemd/system/mysqld.service << 'EOF'
[Unit]
Description=MySQL Server 8.4
Documentation=https://dev.mysql.com/doc/refman/8.4/en/
After=network.target syslog.target

[Service]
User=mysql
Group=mysql
Type=notify
ExecStart=/data/module/mysql/bin/mysqld --defaults-file=/data/module/mysql/conf/my.cnf
LimitNOFILE=65535
LimitNPROC=65535
PrivateTmp=false
Restart=on-failure
RestartSec=5s
TimeoutStartSec=600
TimeoutStopSec=600

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

确认服务文件：

```bash
systemctl cat mysqld
systemctl show mysqld -p FragmentPath -p DropInPaths
```

> 注意：`ExecStart` 使用 `/data/module/mysql/bin/mysqld` 软链接路径。升级时只切换 `/data/module/mysql` 软链接，服务文件通常不需要修改。

---

## 10. 启动与安全加固

### 10.1 启动 MySQL

```bash
systemctl enable mysqld
systemctl start mysqld
systemctl status mysqld --no-pager
```

查看日志：

```bash
tail -100 /data/module/mysql/logs/error.log
ss -lntp | grep ':13306'
```

### 10.2 生成生产环境强密码

生产密码必须随机生成，不要使用人工编造的 `Mysql@123`、`P@ssw0rd`、`Admin@123456`、姓名、项目名、公司名、手机号、生日、常见单词等可猜测内容。

如果企业密码规范要求必须包含大小写字母、数字和特殊字符，推荐使用以下方式生成：

```bash
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:,.~!#$^*-' < /dev/urandom | head -c 48; echo
```

该命令会生成类似 `P@ssw0rd` 风格但真正随机的复杂密码。字符集刻意避开了空格、单引号、双引号、反斜杠、反引号、分号、管道符等容易影响 SQL、Shell 或配置文件解析的字符。

如果现场更重视命令行和配置文件转义安全，且企业密码策略不强制要求特殊字符，也可使用十六进制随机密码：

```bash
openssl rand -hex 32
```

真实密码不要写入文档、提交到 Git 仓库或发送到群聊，应保存到企业密码管理系统或受控密钥管理平台。

### 10.3 修改 root 密码

使用初始化生成的临时密码登录：

```bash
mysql -uroot -p -S /data/module/mysql/run/mysql.sock
```

修改 root 密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'wyz123!@#';
FLUSH PRIVILEGES;
```

> 注意：不要在文档、脚本、工单中保留真实生产密码。上面的密码只是示例，生产必须使用密码管理系统生成并托管。

### 10.4 删除匿名用户和无用账号

```sql
SELECT user, host, account_locked, plugin FROM mysql.user ORDER BY user, host;
DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'%';
FLUSH PRIVILEGES;
```

### 10.5 启用 root 远程登录

确认当前 root 账号：

```sql
SELECT user, host FROM mysql.user WHERE user = 'root';
```

创建或更新远程 root 账号，使 root 可从任意来源登录并具备完整管理权限：

```sql
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY 'wyz123!@#';
ALTER USER 'root'@'%' IDENTIFIED BY 'wyz123!@#';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

> 注意：`'root'@'%'` 表示允许 root 从任意主机远程登录，生产环境必须配合防火墙、安全组、强密码和审计措施使用。真实密码不要写入文档、脚本或工单。

### 10.6 创建业务账号

示例：业务库名为 `appdb`，业务服务器网段为 `10.10.20.%`。

```sql
CREATE DATABASE IF NOT EXISTS appdb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
CREATE USER 'app_user'@'10.10.20.%' IDENTIFIED BY 'ChangeMe_App_Strong_Password_2026!';
GRANT SELECT, INSERT, UPDATE, DELETE, EXECUTE ON appdb.* TO 'app_user'@'10.10.20.%';
FLUSH PRIVILEGES;
```

如业务需要 DDL 权限，建议拆分迁移账号：

```sql
CREATE USER 'app_ddl'@'10.10.20.%' IDENTIFIED BY 'ChangeMe_DDL_Strong_Password_2026!';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP, REFERENCES, EXECUTE ON appdb.* TO 'app_ddl'@'10.10.20.%';
FLUSH PRIVILEGES;
```

**推荐：** 应用运行账号不授予 `CREATE`、`ALTER`、`DROP` 权限；数据库变更通过迁移账号和变更流程执行。

### 10.6 可选：启用密码校验组件

MySQL 8.4 可使用 `validate_password` 组件增强密码策略。

```sql
INSTALL COMPONENT 'file://component_validate_password';
SHOW VARIABLES LIKE 'validate_password%';
SET PERSIST validate_password.policy = 'STRONG';
SET PERSIST validate_password.length = 14;
SET PERSIST validate_password.mixed_case_count = 1;
SET PERSIST validate_password.number_count = 1;
SET PERSIST validate_password.special_char_count = 1;
```

---

## 11. 网络边界与访问控制

### 11.1 确认外层网络边界

本文默认 MySQL 监听所有本机 IPv4 地址：

```ini
bind-address = 0.0.0.0
```

该配置用于满足开发、运维工具、监控和内网服务的远程连接需求。部署前必须确认服务器外层已有防火墙、安全组、VPN、堡垒机或其他网络边界保护，MySQL 端口不得直接暴露到公网。

确认监听状态：

```bash
ss -lntp | grep ':13306'
```

### 11.2 访问来源控制

本文不维护本机 `firewalld` 规则，主机防火墙基线由《CentOS7.9生产环境初始化手册.md》统一规定。部署前需要确认以下事项：

- TCP `13306` 仅对可信内网、办公网、VPN、堡垒机或应用服务器开放；
- 禁止从公网直接访问 TCP `13306`；
- 管理入口与数据库入口应分离授权；
- 访问来源调整应同步记录到资产台账、CMDB 或网络策略记录中。

### 11.3 MySQL 账号来源限制

网络边界之外，MySQL 账号仍应限制来源和权限。开发、运维和业务账号应分别创建，避免共用高权限账号；除非来源网络已被统一收敛，否则不建议给高权限账号配置 `%` 来源。

---

## 12. 开发环境 SQL 快速还原

MySQL 初始化、启动、安全加固和网络访问确认完成后，通常还需要将开发环境导出的业务 SQL 快速还原到新实例中，才能完成真正可用的部署交付。

本节以开发环境导出的 SQL 文件已经上传到服务器为前提，示例路径如下：

```text
/data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql
```

### 12.1 还原前确认

确认 MySQL 服务状态、监听端口和 SQL 文件：

```bash
systemctl is-active mysqld
ss -lntp | grep ':13306'
ls -lh /data/backup/12mysql-20260616/
```

确认 SQL 文件大小、文件类型和开头内容：

```bash
du -sh /data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql
file /data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql
head -50 /data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql
```

重点确认以下内容：

- SQL 文件是否为纯文本 SQL，或是否需要先解压；
- SQL 中是否包含 `CREATE DATABASE` 和 `USE` 语句；
- SQL 中是否包含开发环境不应带入的账号、权限、定时事件或测试库；
- SQL 中是否存在 `DEFINER=`，避免因开发环境账号在新实例不存在导致视图、函数、存储过程或触发器导入失败；
- 目标实例是否为空库，避免误覆盖已有业务数据。

如需快速检查是否存在常见语句：

```bash
grep -n -E '^(CREATE DATABASE|USE |CREATE.*DEFINER|CREATE USER|GRANT |CREATE EVENT)' /data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql | head -50
```

### 12.2 还原方案选择


| 方案         | 命令特点                  | 适用场景                       |
| ---------- | --------------------- | -------------------------- |
| 方案 A：容错导入  | 增加 `--force`，遇到错误继续执行 | 推荐方案，适合源库存在废弃视图、缺失依赖对象或导出时已使用 `--force` 的场景 |
| 方案 B：普通导入  | 不加 `--force`，遇到错误立即停止 | 适合 SQL 文件质量明确、要求遇错立即中止的正式交付场景 |
| 方案 C：压缩包导入 | `gzip -dc` 后管道导入      | 适合 `.sql.gz` 大文件，减少传输和落盘体积 |


**推荐：** 如果 SQL 文件是通过 `--force` 导出的，或源库已知存在废弃视图、缺失底层表、历史过程、事件等不完整对象，恢复时也应使用 `--force` 容错导入。`--force` 会跳过错误继续执行，导入完成后必须检查日志中的 `ERROR` 清单，确认失败对象是否可忽略。

### 12.3 可选：清理半导入业务库

如果前一次导入中断，且确认当前实例只用于本次恢复，可先清理所有业务库：

```bash
read -s -p "MySQL root password: " MYSQL_PWD; echo; export MYSQL_PWD; mysql -uroot -S /data/module/mysql/run/mysql.sock -N -B -e "SELECT CONCAT('DROP DATABASE IF EXISTS \`', SCHEMA_NAME, '\`;') FROM information_schema.SCHEMATA WHERE SCHEMA_NAME NOT IN ('mysql','sys','information_schema','performance_schema');" > /tmp/drop-business-databases.sql && mysql -uroot -S /data/module/mysql/run/mysql.sock < /tmp/drop-business-databases.sql; rm -f /tmp/drop-business-databases.sql; unset MYSQL_PWD
```

> 注意：该命令会删除除 `mysql`、`sys`、`information_schema`、`performance_schema` 之外的所有库，仅限确认无保留业务库的恢复环境使用。

### 12.4 执行容错导入

带库名的 SQL 文件，后台加速导入：

```bash
mkdir -p /data/module/mysql/logs/restore; read -s -p "MySQL root password: " MYSQL_PWD; echo; export MYSQL_PWD; nohup bash -c 'mysql -uroot -S /data/module/mysql/run/mysql.sock -e "SET GLOBAL innodb_flush_log_at_trx_commit=2; SET GLOBAL sync_binlog=0; ALTER INSTANCE DISABLE INNODB REDO_LOG;"; mysql -uroot -S /data/module/mysql/run/mysql.sock --binary-mode=1 --show-warnings --force --init-command="SET SESSION foreign_key_checks=0; SET SESSION unique_checks=0; SET SESSION sql_log_bin=0;" < /data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql > /data/module/mysql/logs/restore/restore-force-$(date +%F-%H%M%S).log 2>&1; mysql -uroot -S /data/module/mysql/run/mysql.sock -e "ALTER INSTANCE ENABLE INNODB REDO_LOG; SET GLOBAL innodb_flush_log_at_trx_commit=1; SET GLOBAL sync_binlog=1;"' > /data/module/mysql/logs/restore/restore-nohup-$(date +%F-%H%M%S).out 2>&1 & unset MYSQL_PWD
```

不带库名的单库 SQL 文件，先创建目标库，再后台导入：

```bash
mkdir -p /data/module/mysql/logs/restore; read -s -p "MySQL root password: " MYSQL_PWD; echo; export MYSQL_PWD; nohup bash -c 'mysql -uroot -S /data/module/mysql/run/mysql.sock -e "SET GLOBAL innodb_flush_log_at_trx_commit=2; SET GLOBAL sync_binlog=0; ALTER INSTANCE DISABLE INNODB REDO_LOG; CREATE DATABASE IF NOT EXISTS appdb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;"; mysql -uroot -S /data/module/mysql/run/mysql.sock --binary-mode=1 --show-warnings --force --init-command="SET SESSION foreign_key_checks=0; SET SESSION unique_checks=0; SET SESSION sql_log_bin=0;" appdb < /data/backup/12mysql-20260616/appdb.sql > /data/module/mysql/logs/restore/restore-force-appdb-$(date +%F-%H%M%S).log 2>&1; mysql -uroot -S /data/module/mysql/run/mysql.sock -e "ALTER INSTANCE ENABLE INNODB REDO_LOG; SET GLOBAL innodb_flush_log_at_trx_commit=1; SET GLOBAL sync_binlog=1;"' > /data/module/mysql/logs/restore/restore-nohup-appdb-$(date +%F-%H%M%S).out 2>&1 & unset MYSQL_PWD
```

压缩 SQL 文件，后台解压并导入：

```bash
mkdir -p /data/module/mysql/logs/restore; read -s -p "MySQL root password: " MYSQL_PWD; echo; export MYSQL_PWD; nohup bash -c 'mysql -uroot -S /data/module/mysql/run/mysql.sock -e "SET GLOBAL innodb_flush_log_at_trx_commit=2; SET GLOBAL sync_binlog=0; ALTER INSTANCE DISABLE INNODB REDO_LOG;"; gzip -dc /data/backup/12mysql-20260616/12mysql-business-all-20260616-force.sql.gz | mysql -uroot -S /data/module/mysql/run/mysql.sock --binary-mode=1 --show-warnings --force --init-command="SET SESSION foreign_key_checks=0; SET SESSION unique_checks=0; SET SESSION sql_log_bin=0;" > /data/module/mysql/logs/restore/restore-force-$(date +%F-%H%M%S).log 2>&1; mysql -uroot -S /data/module/mysql/run/mysql.sock -e "ALTER INSTANCE ENABLE INNODB REDO_LOG; SET GLOBAL innodb_flush_log_at_trx_commit=1; SET GLOBAL sync_binlog=1;"' > /data/module/mysql/logs/restore/restore-nohup-$(date +%F-%H%M%S).out 2>&1 & unset MYSQL_PWD
```

查看后台任务和日志：

```bash
ps -ef | grep '[m]ysql.*12mysql-business-all'
ls -lt /data/module/mysql/logs/restore/ | head
```

> 注意：后台命令会在导入结束后自动恢复 redo/binlog 刷盘参数；如果 MySQL 或服务器异常宕机，建议清库后重新导入。

### 12.5 导入后检查

查看最近一次还原日志：

```bash
ls -lt /data/module/mysql/logs/restore/ | head
```

如日志中存在 `ERROR`，必须先确认原因，不能直接视为还原成功：

```bash
grep -n -E 'ERROR|Warning|warning' /data/module/mysql/logs/restore/restore-*.log | tail -100
```

登录 MySQL 检查库、表和数据量：

```bash
mysql -uroot -p -S /data/module/mysql/run/mysql.sock
```

```sql
SHOW DATABASES;
SELECT table_schema, COUNT(*) AS table_count
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','sys','information_schema','performance_schema')
GROUP BY table_schema
ORDER BY table_schema;
```

查看各业务库数据量概览：

```sql
SELECT table_schema,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','sys','information_schema','performance_schema')
GROUP BY table_schema
ORDER BY size_mb DESC;
```

如果导入文件中包含视图、函数、存储过程、触发器或事件，还应检查：

```sql
SHOW FULL TABLES WHERE Table_type = 'VIEW';
SHOW PROCEDURE STATUS WHERE Db NOT IN ('mysql','sys','information_schema','performance_schema');
SHOW FUNCTION STATUS WHERE Db NOT IN ('mysql','sys','information_schema','performance_schema');
SHOW TRIGGERS;
SHOW EVENTS;
```

### 12.5 导入失败处理

常见失败原因和处理建议：


| 问题            | 现象                        | 处理建议                                     |
| ------------- | ------------------------- | ---------------------------------------- |
| 目标库不存在        | `Unknown database`        | 先创建数据库，或确认 dump 文件是否包含 `CREATE DATABASE` |
| 字符集不一致        | 中文乱码或排序规则错误               | 确认 dump 文件和目标库均使用 `utf8mb4`              |
| `DEFINER` 不存在 | 视图、过程、函数、触发器创建失败          | 导入前清理 `DEFINER`，或先创建对应账号                 |
| 对象已存在         | `Table already exists`    | 确认是否重复导入；必要时清空目标库后重新导入                   |
| 权限不足          | `Access denied`           | 使用 root 本地 socket 导入，或给迁移账号补齐必要权限        |
| 包过大           | `max_allowed_packet` 相关错误 | 临时调大 `max_allowed_packet` 后重试            |


如果是全新部署且导入中途失败，推荐清理目标业务库后重新导入，避免半成品数据继续使用。清理前必须再次确认库名：

```sql
SHOW DATABASES;
DROP DATABASE appdb;
CREATE DATABASE appdb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

### 12.6 完成交付确认

SQL 还原完成后，至少完成以下确认：

- 业务库、表、视图、存储过程、函数、触发器和事件数量符合预期；
- 关键业务表行数与开发环境导出记录一致；
- 应用账号可以从指定来源连接；
- 应用启动后可以完成基础读写；
- 错误日志中无还原相关异常；
- 还原文件、还原日志和部署记录已归档。

---

## 13. 生产验证清单

### 13.1 服务状态验证

```bash
systemctl is-enabled mysqld
systemctl is-active mysqld
systemctl status mysqld --no-pager
ss -lntp | grep ':13306'
```

### 13.2 参数验证

```sql
SHOW VARIABLES WHERE Variable_name IN (
  'version',
  'server_id',
  'bind_address',
  'datadir',
  'socket',
  'character_set_server',
  'collation_server',
  'sql_mode',
  'max_connections',
  'innodb_buffer_pool_size',
  'innodb_redo_log_capacity',
  'innodb_flush_log_at_trx_commit',
  'sync_binlog',
  'log_bin',
  'binlog_format',
  'binlog_expire_logs_seconds',
  'slow_query_log',
  'long_query_time',
  'local_infile',
  'skip_name_resolve',
  'lower_case_table_names'
);
```

### 13.3 安全验证

```sql
SELECT user, host, account_locked, plugin FROM mysql.user ORDER BY user, host;
SHOW GRANTS FOR 'app_user'@'10.10.20.%';
```

确认：

- 不存在匿名账号；
- 不存在 `'root'@'%'`；
- 应用账号只允许指定来源网段；
- 应用运行账号不具备全局 `SUPER`、`SYSTEM_USER`、`FILE` 等高危权限。

### 13.4 日志验证

```sql
SHOW VARIABLES LIKE 'log_error';
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';
SHOW BINARY LOGS;
```

系统侧检查：

```bash
ls -lh /data/module/mysql/logs/
ls -lh /data/module/mysql/binlog/
tail -50 /data/module/mysql/logs/error.log
```

### 13.5 基础读写验证

```sql
CREATE DATABASE IF NOT EXISTS ops_check DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
CREATE TABLE ops_check.t_mysql_check (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  check_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  content VARCHAR(100) NOT NULL
) ENGINE=InnoDB;
INSERT INTO ops_check.t_mysql_check(content) VALUES ('mysql84 production check');
SELECT * FROM ops_check.t_mysql_check;
DROP DATABASE ops_check;
```

### 13.6 重启验证

```bash
systemctl restart mysqld
systemctl is-active mysqld
mysqladmin -uroot -p -S /data/module/mysql/run/mysql.sock ping
```

---

## 14. 运维检查命令

### 14.1 查看实例状态

```sql
SHOW GLOBAL STATUS LIKE 'Uptime';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW GLOBAL STATUS LIKE 'Aborted_connects';
SHOW ENGINE INNODB STATUS\G
```

### 14.2 查看连接来源

```sql
SELECT user, host, db, command, time, state
FROM information_schema.processlist
ORDER BY time DESC
LIMIT 30;
```

### 14.3 查看慢 SQL 配置

```sql
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'slow_query_log_file';
SHOW VARIABLES LIKE 'long_query_time';
```

查看慢日志文件：

```bash
tail -100 /data/module/mysql/logs/slow.log
```

### 14.4 查看 binlog

```sql
SHOW MASTER STATUS;
SHOW BINARY LOGS;
```

> MySQL 8.4 仍兼容 `SHOW MASTER STATUS`，但新脚本建议逐步关注官方术语变化。

### 14.5 查看磁盘空间

```bash
df -hT /data
du -sh /data/module/mysql/*
find /data/module/mysql/binlog -type f -name 'mysql-bin.*' -printf '%TY-%Tm-%Td %TH:%TM %s %p\n' | sort | tail -20
```

### 14.6 在线修改参数

优先使用 `SET PERSIST` 写入持久化参数，但需要纳入配置管理，避免 `/data/module/mysql/conf/my.cnf` 与运行参数长期不一致。

```sql
SET PERSIST long_query_time = 0.5;
SHOW VARIABLES LIKE 'long_query_time';
```

不建议在线随意修改以下参数：

- `lower_case_table_names`；
- `character_set_server`；
- `collation_server`；
- `innodb_page_size`；
- `datadir`；
- `server_id`；
- 日志路径类参数。

---

## 15. 变更与回滚建议

### 15.1 配置变更流程

1. 记录变更目的、影响范围和回滚方案；
2. 备份 `/data/module/mysql/conf/my.cnf`；
3. 在测试环境验证参数；
4. 使用 `mysqld --validate-config` 检查配置；
5. 选择低峰窗口执行；
6. 重启前确认业务连接池重连能力；
7. 重启后验证服务、日志、参数和业务读写。

示例：

```bash
cp /data/module/mysql/conf/my.cnf /data/module/mysql/conf/my.cnf.bak.$(date +%F-%H%M%S)
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --validate-config
systemctl restart mysqld
systemctl status mysqld --no-pager
```

### 15.2 回滚配置

```bash
cp /data/module/mysql/conf/my.cnf.bak.<时间戳> /data/module/mysql/conf/my.cnf
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --validate-config
systemctl restart mysqld
```

### 15.3 Generic 二进制版本升级流程

Generic 二进制部署的升级原则是：**新版本解压到新目录，验证通过后切换软链接，不覆盖旧版本目录**。

升级前准备：

```bash
mysql --version
mysqld --version
readlink -f /data/module/mysql
cp /data/module/mysql/conf/my.cnf /data/module/mysql/conf/my.cnf.bak.$(date +%F-%H%M%S)
```

上传并校验新版本安装包：

```bash
cd /opt/software/mysql
sha256sum -c mysql-8.4.y-linux-glibc2.17-x86_64.tar.xz.sha256
```

解压新版本：

```bash
tar -xJf mysql-8.4.y-linux-glibc2.17-x86_64.tar.xz -C /data/module/
chown -R root:root /data/module/mysql-8.4.y-linux-glibc2.17-x86_64
chmod 755 /data/module/mysql-8.4.y-linux-glibc2.17-x86_64
```

停机窗口内切换：

```bash
systemctl stop mysqld
ln -sfn mysql-8.4.y-linux-glibc2.17-x86_64 /data/module/mysql
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --validate-config
systemctl start mysqld
systemctl status mysqld --no-pager
mysql --version
mysqladmin -uroot -p -S /data/module/mysql/run/mysql.sock ping
```

升级后验证：

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'basedir';
SHOW VARIABLES LIKE 'datadir';
SHOW BINARY LOGS;
```

> 注意：跨大版本升级不能只切换二进制目录，必须按官方升级路径执行兼容性检查、备份和升级验证。MySQL 8.4 LTS 补丁版本内升级也应先在测试环境验证。

### 15.4 Generic 二进制版本回滚流程

回滚前必须确认数据文件没有发生不可逆升级。补丁版本回滚也需要谨慎评估，尤其是已经启动过新版本并执行过写入的场景。

查看当前版本与旧版本目录：

```bash
readlink -f /data/module/mysql
ls -ld /data/module/mysql-8.4.*
```

停机并切回旧版本：

```bash
systemctl stop mysqld
ln -sfn mysql-8.4.9-linux-glibc2.17-x86_64 /data/module/mysql
cp /data/module/mysql/conf/my.cnf.bak.<时间戳> /data/module/mysql/conf/my.cnf
mysqld --defaults-file=/data/module/mysql/conf/my.cnf --validate-config
systemctl start mysqld
systemctl status mysqld --no-pager
mysql --version
```

回滚后验证：

```bash
mysqladmin -uroot -p -S /data/module/mysql/run/mysql.sock ping
tail -100 /data/module/mysql/logs/error.log
```

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'basedir';
SHOW VARIABLES LIKE 'datadir';
```

### 15.5 版本资产登记建议

Generic 二进制安装不进入 RPM 数据库，因此团队必须维护资产记录。建议至少登记：


| 字段       | 示例                                                                      |
| -------- | ----------------------------------------------------------------------- |
| 主机名      | `mysql-prod-01`                                                         |
| 内网 IP    | `10.10.10.15`                                                           |
| MySQL 版本 | `8.4.9`                                                                 |
| 安装包名称    | `mysql-8.4.9-linux-glibc2.17-x86_64.tar.xz`                             |
| SHA256   | `<实际校验值>`                                                               |
| 当前软链接    | `/data/module/mysql -> /data/module/mysql-8.4.9-linux-glibc2.17-x86_64` |
| 配置文件     | `/data/module/mysql/conf/my.cnf`                                        |
| 数据目录     | `/data/module/mysql/data`                                               |
| 端口       | `13306`                                                                 |
| 部署人      | `<姓名>`                                                                  |
| 部署时间     | `<YYYY-MM-DD HH:MM>`                                                    |
| 回滚版本     | `mysql-8.4.w-linux-glibc2.17-x86_64`                                    |


### 15.6 备份建议

单体生产库至少应具备以下备份能力：


| 类型        | 建议      | 说明                                                         |
| --------- | ------- | ---------------------------------------------------------- |
| 全量备份      | 每日一次    | 可使用 MySQL Enterprise Backup、Percona XtraBackup 或 mysqldump |
| binlog 备份 | 实时或定时同步 | 支持时间点恢复                                                    |
| 备份保留      | 7~30 天  | 结合容量和合规要求                                                  |
| 恢复演练      | 每月至少一次  | 只备份不演练等于没有备份                                               |


`mysqldump` 小库备份示例：

```bash
mkdir -p /data/module/mysql/backup/$(date +%F)
mysqldump -uroot -p \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --set-gtid-purged=OFF \
  --all-databases \
  | gzip > /data/module/mysql/backup/$(date +%F)/all-databases.sql.gz
```

> 注意：大库不建议依赖 `mysqldump` 作为唯一备份方式，应使用物理热备工具并定期做恢复演练。

---

## 16. 常见问题

### Q1: CentOS 7.9 还能部署 MySQL 8.4 吗？

可以在存量环境中部署，但不建议作为全新生产系统的首选。CentOS 7 已 EOL，安全补丁、内核、glibc 和生态支持都会逐步受限。全新生产环境建议选择 EL9 系发行版。

### Q2: 是否可以把 `bind-address` 设置为 `0.0.0.0`？

可以，但前提是 MySQL 端口不直接暴露到公网，并且服务器外层已有防火墙、安全组、VPN、堡垒机或统一网络策略保护。本文默认使用 `0.0.0.0`，用于满足开发、运维工具、监控和内网服务的远程连接需求；安全边界由外层网络策略和 MySQL 账号来源限制共同承担。

### Q3: 为什么必须开启 binlog？

binlog 是误操作恢复、增量恢复、审计、数据订阅和后续复制的基础。即使当前是单体部署，也建议开启。不开 binlog 会降低恢复能力。

### Q4: `innodb_buffer_pool_size` 是否越大越好？

不是。还需要给连接线程、临时表、排序、操作系统页缓存、备份工具和运维命令保留内存。过大可能导致 OOM 或 Swap，反而影响稳定性。

### Q5: 是否需要关闭 Swap？

有三种方案：


| 方案                             | 说明             | 建议           |
| ------------------------------ | -------------- | ------------ |
| 方案 A：完全关闭 Swap                 | 避免数据库页被换出      | 内存充足、参数合理时可用 |
| 方案 B：保留 Swap，`vm.swappiness=1` | 兼顾兜底和低换出倾向     | 常规生产推荐       |
| 方案 C：保留默认 swappiness           | 系统可能更积极使用 Swap | 不推荐          |


**推荐：** 数据库服务器保留少量 Swap 并设置 `vm.swappiness=1`。若业务对延迟极敏感且内存容量充分，可经压测后关闭 Swap。

### Q6: 是否需要运行 `mysql_secure_installation`？

可以，但生产环境更推荐显式执行本文中的 SQL 加固步骤。这样每一步可审计、可复查、可纳入自动化流程。

### Q7: 为什么不建议应用账号拥有 DDL 权限？

应用运行账号如果拥有 `DROP`、`ALTER` 等权限，SQL 注入、程序缺陷或误操作可能直接破坏表结构。生产建议运行账号和迁移账号分离。

### Q8: MySQL 8.4 默认认证插件是否需要改为 `mysql_native_password`？

不建议为了兼容旧客户端而全局降级认证插件。应优先升级客户端驱动，使其支持 MySQL 8 默认认证方式。只有在短期兼容过渡期，才针对特定账号评估降级，并制定退出计划。

---

## 参考链接

- MySQL 8.4 Reference Manual: [https://dev.mysql.com/doc/refman/8.4/en/](https://dev.mysql.com/doc/refman/8.4/en/)
- MySQL 8.4 Release Notes: [https://dev.mysql.com/doc/relnotes/mysql/8.4/en/](https://dev.mysql.com/doc/relnotes/mysql/8.4/en/)
- MySQL Yum Repository: [https://dev.mysql.com/downloads/repo/yum/](https://dev.mysql.com/downloads/repo/yum/)
- Installing MySQL on Linux Using the MySQL Yum Repository: [https://dev.mysql.com/doc/refman/8.4/en/linux-installation-yum-repo.html](https://dev.mysql.com/doc/refman/8.4/en/linux-installation-yum-repo.html)
- Installing MySQL on Unix/Linux Using Generic Binaries: [https://dev.mysql.com/doc/refman/8.4/en/binary-installation.html](https://dev.mysql.com/doc/refman/8.4/en/binary-installation.html)

