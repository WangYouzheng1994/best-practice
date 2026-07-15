# TDengine 3.4 OSS 单体生产级配置部署手册

> **Author:** 王有政<br>
> **日期:** 2026-07-15<br>
> **系统:** CentOS 7.9 x86_64<br>
> **版本:** TDengine TSDB-OSS 3.4.1.13<br>
> **范围:** 单体部署。集群、高可用和跨机房容灾不在本文展开。

---

## 使用说明

本文按实际部署顺序编写。第一次部署时，从第 1 节开始依次执行。

本文已经固定安装方式、目录、端口和组件，执行人员按顺序操作即可：

- 使用官方 Linux-Generic x64 压缩包；
- 程序安装到带版本目录 `/data/module/tdengine/3.4.1.13/taos`；
- 当前版本软链接为 `/data/module/tdengine/current`；
- 数据、日志和临时文件放在 `/data/module/tdengine`；
- 使用团队自定义端口 `16030`、`16041`、`16043`、`16060`，不使用官方默认端口；
- 启动 `taosd`、`taosadapter`、`taoskeeper`、`taos-explorer`；
- Explorer 只向运维网段开放；
- 使用 `taosdump` 做逻辑备份；
- 使用 taosKeeper `/metrics` 接入 Prometheus。

执行约定：

1. 使用 root Shell 执行：`sudo -i`；
2. 文中的 `<...>` 必须替换成现场值；
3. 密码不得写入 Shell 命令、Git、工单正文或聊天记录；
4. 已经运行 TDengine 或存在历史数据时，不要使用本文“首次安装”步骤覆盖安装；
5. 集群部署不要套用本文；
6. 任一命令报错立即停止，确认原因并修正后再继续。

---

## 1. 进入 root Shell

```bash
sudo -i
```

---

## 2. 检查服务器

### 2.1 系统和资源

```bash
cat /etc/redhat-release
uname -m
nproc
free -h
df -hT /data
chronyc tracking
```

继续部署前必须满足：

- CentOS 7.9、x86_64；
- 时间同步正常；
- `/data` 是已确认的数据盘；
- 推荐生产起步规格为 16C/64G；
- 数据盘按容量评审准备，并至少保留 30% 空闲空间；
- 已完成《CentOS7.9生产环境初始化手册.md》的全部公共基线。

确认初始化手册已经安装本文需要的基础命令：

```bash
command -v vim wget openssl curl iostat
```

预期依次输出五个命令路径，不得缺少任何一行。

### 2.2 检查历史 TDengine

```bash
rpm -qa | grep -Ei 'tdengine|taos' || true
command -v taos taosd taosadapter taoskeeper taos-explorer 2>/dev/null || true
systemctl list-unit-files | grep -Ei '^taos' || true
ps -ef | grep -E '[t]aosd|[t]aosadapter|[t]aoskeeper|[t]aos-explorer' || true
ss -lntup | grep -E ':(6030|6041|6043|6044|6045|6046|6047|6048|6049|6060|16030|16041|16043|16060)\b' || true
find /etc/taos /var/lib/taos /var/log/taos -maxdepth 2 -ls 2>/dev/null
```

预期结果是：没有已安装的 TDengine、没有相关进程、端口未占用、没有未知数据。

只要发现历史安装或数据，立即停止首次安装，先确认版本、数据目录、备份和业务状态。

---

## 3. 确认 hostname 和 FQDN

本文默认保留服务器现有静态 hostname，不无条件修改。

```bash
hostnamectl --static
hostname -f
getent ahostsv4 "$(hostnamectl --static)" || true
```

hostname 必须唯一、稳定，不包含空格和多余点号。

如果上面的 `static` 是空值、`localhost`、`localhost.localdomain`，或不符合企业命名规范，必须在首次安装前修改。已有合格 hostname 时不要改。

需要修改时执行，并把 `<新hostname>` 换成正式名称：

```bash
hostnamectl set-hostname <新hostname>
hostnamectl --static
```

如果 `getent` 没有返回服务器真实内网 IP，先备份再编辑 `/etc/hosts`：

```bash
cp -a /etc/hosts "/etc/hosts.$(date +%F_%H%M%S).bak"
vim /etc/hosts
```

无企业内网 DNS 时加入：

```text
<服务器内网IP> <当前hostname>
```

例如：

```text
10.10.10.21 semi-idc-huagong-demo01
```

**硬性要求：TDengine 使用的 hostname 只能解析到服务器真实内网 IP，不能指向 IPv4 回环地址 `127.0.0.1`、`127.0.1.1`，也不能指向 IPv6 回环地址 `::1`。这些地址都只代表当前机器自己。**

如果原文件已经把当前 hostname 写在回环地址后面，必须删除该 hostname，只保留 `localhost` 等系统回环名称。例如下面这种配置是错误的：

```text
127.0.0.1 semi-idc-huagong-demo01
::1 semi-idc-huagong-demo01
```

不要使用 `echo >> /etc/hosts` 重复追加。

保存后验证：

```bash
getent ahostsv4 "$(hostnamectl --static)"
getent hosts "$(hostnamectl --static)"
getent ahostsv4 "$(hostnamectl --static)" | awk '{print $1}' | sort -u
grep -E '^(127\.[0-9.]+|::1)[[:space:]]' /etc/hosts | grep -Fw "$(hostnamectl --static)" || true

# 现场允许 ICMP 时执行
ping -c 3 "$(hostnamectl --static)"
```

`getent` 只能返回真实内网 IP，最后一条回环地址检查必须没有输出。

TDengine 初始化完成后，不要直接修改 hostname 或 `fqdn`。

---

## 4. 做容量确认

部署前至少确认以下四个值：

| 项目 | 现场值 |
|---|---|
| 每秒写入记录数 | `<填写>` |
| 单条记录估算字节数 | `<填写>` |
| 数据保留天数 | `<填写>` |
| 峰值并发查询数 | `<填写>` |

快速估算：

```text
日原始数据量 = 每秒记录数 × 单条字节数 × 86400
保留期原始量 = 日原始数据量 × 保留天数
数据文件估算 = 保留期原始量 ÷ 实测压缩比
数据盘最低容量 = 数据文件估算 ÷ 0.70
```

压缩比必须用代表性数据实测。没有实测数据时，可以按 5 倍压缩做初步预算，但不能当作容量承诺。

如果估算后单机长期 CPU、内存或磁盘无法保持 30% 以上余量，应停止单体部署并评估集群。

---

## 5. 创建目录

```bash
mkdir -p /data/module/tdengine/3.4.1.13/install
mkdir -p /data/module/tdengine/{data,explorer/data,logs/taosd,logs/adapter,logs/keeper,logs/explorer,tmp}
mkdir -p /data/backup/tdengine

chown -R root:root /data/module/tdengine /data/backup/tdengine
chmod 0755 /data/module/tdengine /data/module/tdengine/3.4.1.13 /data/module/tdengine/3.4.1.13/install
chmod 0750 /data/module/tdengine/data /data/module/tdengine/explorer /data/module/tdengine/explorer/data /data/module/tdengine/logs /data/module/tdengine/logs/* /data/module/tdengine/tmp /data/backup/tdengine

ls -ld \
  /data/module/tdengine \
  /data/module/tdengine/3.4.1.13/install \
  /data/module/tdengine/data \
  /data/module/tdengine/explorer/data \
  /data/module/tdengine/logs \
  /data/module/tdengine/tmp \
  /data/backup/tdengine
```

预期：以上目录全部存在；`install` 权限为 `755`，data、logs、tmp 和 backup 相关目录权限为 `750`，属主均为 `root:root`。

目录用途：

| 路径 | 用途 |
|---|---|
| `/data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz` | 安装包 |
| `/data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13` | 解压目录 |
| `/data/module/tdengine/3.4.1.13/taos` | 程序版本目录 |
| `/data/module/tdengine/current` | 当前版本软链接 |
| `/data/module/tdengine/data` | 数据和 WAL |
| `/data/module/tdengine/explorer/data` | Explorer 本地数据 |
| `/data/module/tdengine/logs/*` | 各组件日志 |
| `/data/module/tdengine/tmp` | 临时文件 |
| `/data/backup/tdengine` | 本机备份暂存 |

`/data/backup` 是本机备份目录，备份完成后仍需复制到其他主机或独立备份存储。

---

## 6. 下载并校验安装包

生产环境推荐从企业制品库上传到上述安装包路径。如果当前服务器允许访问官方 CDN，可以执行：

```bash
wget -O /data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz \
  https://downloads.taosdata.com/tdengine-tsdb-oss/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz
```

校验文件：

```bash
test "$(stat -c '%s' /data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz)" -eq 300274656
echo '4c0fc6e31d59107a5125d5284fc58c32  /data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz' | md5sum -c -
echo '1b8cba22f1ea510ec72863041413ad86970ece2501d15c6eff482db189add515  /data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz' | sha256sum -c -
gzip -t /data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz
```

MD5、SHA256 必须显示 `OK`，`gzip -t` 必须没有错误输出。

上面的 SHA256 是 2026-07-13 对官方 CDN 文件的实算值，不是厂商数字签名。企业制品库仍应保存自己的 SHA256 清单和下载审计记录。

---

## 7. 解压并安装

### 7.1 解压

```bash
mkdir -p /data/module/tdengine/3.4.1.13/install
ls -ld /data/module/tdengine/3.4.1.13/install

tar -zxf /data/module/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz \
  -C /data/module/tdengine/3.4.1.13/install

ls -lh \
  /data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13/install.sh \
  /data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13/package.tar.gz
```

必须实际看到 `install.sh` 和 `package.tar.gz`，否则停止安装。

内层 `package.tar.gz` 由 `install.sh` 自动处理，不要手工解压。

### 7.2 安装

```bash
cd /data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13
/data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13/install.sh \
  -v server \
  -e no \
  -d /data/module/tdengine/3.4.1.13
```

注意：`-d` 指定的是父目录，安装器会自动追加 `/taos`。最终目录应为：

```text
/data/module/tdengine/3.4.1.13/taos
```

预期：安装器显示 `TDengine TSDB has been installed successfully!`，且没有 `error` 或 `failed`。安装器会自动启用尚未配置的 Explorer，下一步先将全部服务停下。

### 7.3 安装后处理

安装器会启用 Explorer，并可能把 `/var/log/taos` 权限设置为 `777`。先停止全部服务，Explorer 配置完成前暂时禁止开机启动：

```bash
systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
systemctl disable taos-explorer
chmod 0750 /var/log/taos

systemctl is-enabled taos-explorer || true
stat -c '%a %U:%G %n' /var/log/taos
```

预期：

```text
disabled
750 root:root /var/log/taos
```

创建当前版本软链接，并把数据、日志指向生产目录：

```bash
ln -sfnT /data/module/tdengine/3.4.1.13/taos /data/module/tdengine/current
ln -sfnT /data/module/tdengine/data /data/module/tdengine/3.4.1.13/taos/data
ln -sfnT /data/module/tdengine/logs/taosd /data/module/tdengine/3.4.1.13/taos/log

ls -ld /data/module/tdengine/current
ls -ld /data/module/tdengine/3.4.1.13/taos/data
ls -ld /data/module/tdengine/3.4.1.13/taos/log
```

预期三个软链接的目标分别为：

```text
/data/module/tdengine/current -> /data/module/tdengine/3.4.1.13/taos
/data/module/tdengine/3.4.1.13/taos/data -> /data/module/tdengine/data
/data/module/tdengine/3.4.1.13/taos/log -> /data/module/tdengine/logs/taosd
```

### 7.4 验证安装结果

```bash
test -x /data/module/tdengine/3.4.1.13/taos/bin/taosd && echo 'PASS: taosd 可执行文件存在'
cat /data/module/tdengine/3.4.1.13/taos/.install_path

command -v taos taosd taosdump taosadapter taoskeeper
readlink -f "$(command -v taosd)"
taos -V
taosd -V
taosdump -V

systemctl cat taosd | grep '^ExecStart='
systemctl cat taosadapter | grep '^ExecStart='
systemctl cat taoskeeper | grep '^ExecStart='
systemctl is-enabled taosd taosadapter taoskeeper
systemctl is-enabled taos-explorer || true
systemctl is-active taosd taosadapter taoskeeper taos-explorer || true
```

首次安装时，本节命令全部必须执行：

| 命令 | 作用 | 正确结果 |
|---|---|---|
| `test -x /data/module/tdengine/3.4.1.13/taos/bin/taosd && echo 'PASS: taosd 可执行文件存在'` | 检查 taosd 文件存在且可以执行 | 显示 `PASS: taosd 可执行文件存在` |
| `cat /data/module/tdengine/3.4.1.13/taos/.install_path` | 检查安装器记录的实际程序目录 | 输出 `/data/module/tdengine/3.4.1.13/taos` |
| `command -v taos taosd taosdump taosadapter taoskeeper` | 检查五个 TDengine 命令已加入系统命令路径 | 依次输出 `/usr/bin/taos`、`/usr/bin/taosd`、`/usr/bin/taosdump`、`/usr/bin/taosadapter`、`/usr/bin/taoskeeper`，不得缺少任何一行 |
| `readlink -f "$(command -v taosd)"` | 穿透 `/usr/bin` 软链接，检查 taosd 实际版本目录 | 输出 `/data/module/tdengine/3.4.1.13/taos/bin/taosd` |
| `taos -V` | 检查客户端命令可以运行及版本 | 输出包含 `3.4.1.13` |
| `taosd -V` | 检查服务端命令可以运行及版本 | 输出包含 `3.4.1.13` |
| `taosdump -V` | 检查备份恢复工具可以运行 | 正常输出 taosdump 版本，不得出现找不到命令或动态库错误 |
| `systemctl cat taosd \| grep '^ExecStart='` | 检查 taosd 的 systemd 启动命令 | `ExecStart` 指向 `/data/module/tdengine/3.4.1.13/taos/bin/taosd`，配置文件为 `/etc/taos/taos.cfg` |
| `systemctl cat taosadapter \| grep '^ExecStart='` | 检查 taosAdapter 的 systemd 启动命令 | `ExecStart` 指向 `/data/module/tdengine/3.4.1.13/taos/bin/taosadapter`，配置文件为 `/etc/taos/taosadapter.toml` |
| `systemctl cat taoskeeper \| grep '^ExecStart='` | 检查 taosKeeper 的 systemd 启动命令 | `ExecStart` 指向 `/data/module/tdengine/3.4.1.13/taos/bin/taoskeeper`，配置文件为 `/etc/taos/taoskeeper.toml` |
| `systemctl is-enabled taosd taosadapter taoskeeper` | 检查三个基础服务是否开机启动 | 连续输出三行 `enabled` |
| `systemctl is-enabled taos-explorer \|\| true` | 确认 Explorer 已禁止开机启动 | 输出 `disabled` |
| `systemctl is-active taosd taosadapter taoskeeper taos-explorer \|\| true` | 确认配置完成前服务没有提前启动 | 连续输出四行 `inactive` |

`disabled` 和 `inactive` 在 systemd 中会返回非零状态，因此命令末尾使用 `|| true`；这里的输出符合预期，不代表执行失败。

任一结果不符合表格要求，停止后续配置，先检查安装目录、软链接和 systemd unit。

### 7.5 配置环境变量

按 CentOS 初始化手册的公共约定，TDengine 只向 `/etc/profile.d/myenv.sh` 追加自己的环境变量。

编辑团队环境变量文件；如果已经存在其他软件变量，只追加 TDengine 内容，不得覆盖整个文件：

```bash
vim /etc/profile.d/myenv.sh
```

追加：

```bash
# TDENGINE_HOME：TDengine 当前版本软链接，升级时只调整软链接
export TDENGINE_HOME=/data/module/tdengine/current
export PATH=$TDENGINE_HOME/bin:$PATH
```

保存并加载：

```bash
source /etc/profile.d/myenv.sh

echo "$TDENGINE_HOME"
which taos taosd taosdump
```

预期：

```text
/data/module/tdengine/current
/data/module/tdengine/current/bin/taos
/data/module/tdengine/current/bin/taosd
/data/module/tdengine/current/bin/taosdump
```

systemd 使用绝对 `ExecStart`，不依赖交互式 Shell 环境变量。

### 7.6 服务及功能

| 服务 | 功能 | 配置文件 | 端口 | 启动条件 |
|---|---|---|---:|---|
| `taosd` | TDengine 核心数据库服务，负责用户认证、数据写入、存储和查询 | `/etc/taos/taos.cfg` | `16030` | 必须最先启动；启动后才能登录并修改密码 |
| `taosadapter` | REST 和 WebSocket 网关，把应用请求转给 `taosd` | `/etc/taos/taosadapter.toml` | `16041` | 不保存固定的 root 密码，`taosd` 启动后即可启动 |
| `taoskeeper` | 使用数据库账号连接 TDengine，接收监控指标并写入监控数据库 | `/etc/taos/taoskeeper.toml` | `16043` | 必须先修改 root 密码，再把新密码写入配置文件 |
| `taos-explorer` | Web 管理界面，通过 `taosadapter` 访问 TDengine | `/etc/taos/explorer.toml` | `16060` | 修改默认密码后启动，只向运维网段开放 |

启动顺序：`taosd` → `taosadapter` → 修改数据库密码并配置 `taoskeeper` → `taoskeeper` → `taos-explorer`。

Java 应用连接端口：

| 端口 | Java 用途 | 连接方式 |
|---:|---|---|
| `16030` | Native JDBC | `jdbc:TAOS://<TDengine hostname>:16030/<数据库名>`，驱动类为 `com.taosdata.jdbc.TSDBDriver`；应用服务器必须安装 TDengine 原生客户端库 |
| `16041` | JDBC WebSocket，生产推荐 | `jdbc:TAOS-WS://<TDengine hostname>:16041/<数据库名>`，驱动类为 `com.taosdata.jdbc.ws.WebSocketDriver`；应用服务器不需要安装 TDengine 原生客户端库 |
| `16043` | taosKeeper 健康检查和 Prometheus 指标 | 不是 JDBC 端口；Java 业务应用不连接 |
| `16060` | taosExplorer 管理页面 | 不是 JDBC 端口；使用浏览器访问 |

Java 业务应用统一推荐使用 `16041`。Spring Boot 示例：

```properties
spring.datasource.url=jdbc:TAOS-WS://<TDengine hostname>:16041/<数据库名>
spring.datasource.username=<用户名>
spring.datasource.password=<密码>
spring.datasource.driver-class-name=com.taosdata.jdbc.ws.WebSocketDriver
```

账号密码分别放在应用配置的用户名和密码字段中，不要拼进 JDBC URL。

---

## 8. 配置 taosd

备份配置：

```bash
cp -a /etc/taos/taos.cfg "/etc/taos/taos.cfg.$(date +%F_%H%M%S).bak"
vim /etc/taos/taos.cfg
```

在文件中查找并保证以下每个参数只有一个有效值。把 `<当前hostname>` 替换为 `hostnamectl --static` 的输出：

```ini
firstEp             <当前hostname>:16030
fqdn                <当前hostname>
serverPort          16030

dataDir             /data/module/tdengine/data
logDir              /data/module/tdengine/logs/taosd
tempDir             /data/module/tdengine/tmp

minimalDataDirGB    20
minimalLogDirGB     5
minimalTmpDirGB     5

asyncLog            1
numOfLogLines       10000000
logKeepDays         30

telemetryReporting  0
crashReporting      0

monitor             1
monitorFqdn         <当前hostname>
monitorPort         16043
monitorInterval     30
```

安装脚本可能已经写入 `monitorFQDN`。必须删除重复项，统一保留一个 `monitorFqdn`。

检查配置：

```bash
grep -niE '^(firstEp|fqdn|serverPort|dataDir|logDir|tempDir|minimalDataDirGB|minimalLogDirGB|minimalTmpDirGB|asyncLog|numOfLogLines|logKeepDays|telemetryReporting|crashReporting|monitor|monitorFqdn|monitorPort|monitorInterval)[[:space:]]' \
  /etc/taos/taos.cfg
```

确认输出没有重复参数，并且 hostname、端口和目录正确。

---

## 9. 配置 taosAdapter 和 taosExplorer

### 9.1 配置 taosAdapter

备份并编辑：

```bash
cp -a /etc/taos/taosadapter.toml "/etc/taos/taosadapter.toml.$(date +%F_%H%M%S).bak"
vim /etc/taos/taosadapter.toml
```

确认或修改以下已有参数：

```toml
debug = false
port = 16041
taosConfigDir = "/etc/taos"

[cors]
allowAllOrigins = false

[log]
path = "/data/module/tdengine/logs/adapter"
level = "info"
keepDays = 30
rotationCount = 30
rotationSize = "1GB"
compress = true
reservedDiskSize = "5GB"

[uploadKeeper]
enable = true
url = "http://127.0.0.1:16043/adapter_report"
interval = "15s"
timeout = "5s"
retryTimes = 3
retryInterval = "5s"
```

如果文件中存在以下入口，保持关闭：

```toml
[collectd]
enable = false

[statsd]
enable = false

[opentsdb_telnet]
enable = false

[opentsdb]
enable = false

[influxdb]
enable = false

[prometheus]
enable = false
```

以上是 taosAdapter 的协议入口，未经过业务评审不得启用；不影响 taosKeeper 的 `/metrics`。不要追加包内不存在的配置段，只修改实际文件中已有参数。

### 9.2 配置 taosExplorer

Explorer 是随安装包提供的 Web 管理界面，通过本机 taosAdapter 访问 TDengine。备份并编辑：

```bash
cp -a /etc/taos/explorer.toml "/etc/taos/explorer.toml.$(date +%F_%H%M%S).bak"
vim /etc/taos/explorer.toml
```

确认或修改以下参数，把 `<当前hostname>` 替换为 `hostnamectl --static` 的输出：

```toml
port = 16060
addr = "0.0.0.0"
data_dir = "/data/module/tdengine/explorer/data"
cluster = "http://<当前hostname>:16041"
cors = true

[log]
path = "/data/module/tdengine/logs/explorer"
level = "info"
compress = true
rotationCount = 30
rotationSize = "1GB"
reservedDiskSize = "5GB"
keepDays = 30

[monitor]
fqdn = "<当前hostname>"
port = 16043
interval = 10
```

按团队约定，Explorer 使用 hostname 连接 taosAdapter 和 taosKeeper，并开启 CORS。taosAdapter 的 `allowAllOrigins` 仍保持 `false`。

> **端口备注：** Explorer 监听 `0.0.0.0:16060`，由服务器外层防火墙、网络 ACL 或安全设备限制为仅运维网段可访问；本手册不要求为此单独启用本机 firewalld。

跳过产品注册，直接使用 TDengine 数据库账号登录：

```bash
vim /etc/default/taos-explorer
```

写入：

```text
EXPLORER_SKIP_REGISTER=true
```

该参数只跳过 Explorer 产品注册，不会跳过 TDengine 数据库账号认证。

---

## 10. 启动 taosd 和 taosAdapter

本节只启动前两个服务：先用 `taosd` 完成数据库初始化和密码修改；`taosadapter` 不需要预先保存 root 密码，可以同时投入本机验证。`taoskeeper` 必须使用有效的数据库账号和密码连接 TDengine，因此留到第 11 节配置完成后启动。

```bash
systemctl daemon-reload
systemctl enable --now taosd
systemctl enable --now taosadapter

systemctl status taosd --no-pager -l
systemctl status taosadapter --no-pager -l
ss -lntp | grep -E ':(16030|16041)\b'
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:16041/-/ping
journalctl -u taosd -u taosadapter --since '-10 min' --no-pager
```

必须看到：

- `taosd` 和 `taosadapter` 为 `active (running)`；
- `16030/TCP`、`16041/TCP` 正在监听；
- 健康检查输出 `HTTP 200`；
- 日志没有持续报错。

不要执行 `start-all.sh`。它会跳过本手册的分阶段启动顺序，使 Explorer 在默认密码修改前启动，也无法保证 `taoskeeper` 已经写入新密码。这里逐个使用 `systemctl`。

---

## 11. 修改默认 root 密码并启动 taosKeeper、taosExplorer

### 11.1 准备 root 新密码

TDengine 初始管理员账号为 `root`，默认密码 `taosdata` 是公开信息，生产环境必须在开放业务端口前修改。下面的命令只生成一段随机密码，并不会修改 TDengine；生成结果将在 11.2 节设置给 root，并在 11.3 节写入 `taoskeeper` 配置。

同一个新密码需要完成两个不同动作：

| 动作 | 作用 |
|---|---|
| 执行 `ALTER USER root PASS` | 修改 TDengine 数据库中 root 账号真正认可的密码 |
| 写入 `taoskeeper.toml` | 告诉 taosKeeper 后台启动时使用哪个密码登录 TDengine |

只修改其中一处都会导致密码不一致：只写配置文件，数据库仍只认可旧密码；只执行 `ALTER USER`，taosKeeper 不知道新密码。

```bash
openssl rand -base64 24
```

把输出保存到企业密码库，不要写入工单、聊天记录或脚本。

### 11.2 修改密码

```bash
taos -h 127.0.0.1 -P 16030 -u root -p
```

首次输入默认密码：

```text
taosdata
```

进入 CLI 后执行，手工粘贴密码库中的新密码：

```sql
ALTER USER root PASS '<新密码>';
QUIT;
```

重新登录验证：

```bash
taos -h 127.0.0.1 -P 16030 -u root -p
```

### 11.3 配置 taosKeeper

```bash
cp -a /etc/taos/taoskeeper.toml "/etc/taos/taoskeeper.toml.$(date +%F_%H%M%S).bak"
vim /etc/taos/taoskeeper.toml
```

安装器通常已经把 `[tdengine]` 下的 `host` 写成当前 hostname；值正确时直接保留，不要改成 `127.0.0.1`。下面的 `<当前hostname>` 表示 `hostnamectl --static` 的输出：

```toml
instanceId = 64
host = "0.0.0.0"
port = 16043
RotationInterval = "15s"

[tdengine]
host = "<当前hostname>"
port = 16041
username = "root"
password = "<新密码>"
usessl = false

[metrics.database]
name = "log"

[metrics.database.options]
vgroups = 1
buffer = 64
keep = 90
cachemodel = "both"

[log]
path = "/data/module/tdengine/logs/keeper"
level = "info"
rotationCount = 7
keepDays = 30
rotationSize = "1GB"
compress = true
reservedDiskSize = "5GB"
```

`password = "<新密码>"` 必须在 `vim` 中替换为刚刚设置给 root 的真实密码，不能保留占位符。taosKeeper 是 TDengine 客户端，需要保存可用于自动登录的密码；不要把密码写进 Shell 命令、脚本、Git 或工单。

顶层 `host = "0.0.0.0"` 是 taosKeeper 自己的监听地址；`[tdengine]` 下的 `host = "<当前hostname>"` 是 taosKeeper 访问 taosAdapter 的目标地址，两者不能混用。`16043` 由外层防火墙或网络 ACL 仅向监控网段和运维网段开放。

限制配置文件权限：

```bash
chown root:root /etc/taos/taoskeeper.toml
chmod 0600 /etc/taos/taoskeeper.toml
```

启动并验证：

```bash
systemctl enable --now taoskeeper
systemctl status taoskeeper --no-pager -l
ss -lntp | grep ':16043\b'
curl -fsS http://127.0.0.1:16043/check_health
curl -fsS http://127.0.0.1:16043/metrics | sed -n '1,30p'
```

必须看到 taosKeeper 为 `active (running)`、`16043/TCP` 正在监听、健康检查返回 `3.4.1.13`，并且指标输出包含 `# HELP`、`# TYPE` 等 Prometheus 格式内容。

### 11.4 启动 taosExplorer

```bash
systemctl enable --now taos-explorer
systemctl status taos-explorer --no-pager -l
ss -lntp | grep ':16060\b'
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:16060/
```

必须看到 Explorer 为 `active (running)`、`16060/TCP` 正在监听、健康检查输出 `HTTP 200`。浏览器通过 `http://<TDengine服务器内网IP>:16060` 访问，使用 TDengine 数据库账号和已经修改的新密码登录。

---

## 12. 配置端口和防火墙

### 12.1 本文使用的端口

| 端口 | 用途 | 开放范围 |
|---:|---|---|
| `16030/TCP` | taosd、客户端连接 | 业务应用网段和运维网段 |
| `16041/TCP` | taosAdapter、应用连接 | 业务应用网段和运维网段 |
| `16043/TCP` | taosKeeper、Prometheus | 监控网段和运维网段 |
| `16060/TCP` | taosExplorer 管理界面 | 仅运维网段 |

端口已经登记到《服务器端口统一规划规范.md》。数据库端口不得对公网开放。

网络开通申请统一填写：

| 来源 | 目标 | 协议和端口 |
|---|---|---|
| 业务应用网段 | `<TDengine服务器内网IP>` | TCP `16030`、`16041` |
| Prometheus 监控网段 | `<TDengine服务器内网IP>` | TCP `16043` |
| 运维网段 | `<TDengine服务器内网IP>` | TCP `16060` |

不申请官方默认端口，不申请 `16030-16060` 整段端口。

### 12.2 检查 firewalld

```bash
systemctl is-active firewalld 2>/dev/null || true
systemctl is-enabled firewalld 2>/dev/null || true
```

仓库服务器初始化基线默认由外层防火墙、硬件防火墙或统一 ACL 控制，firewalld 通常为关闭状态。此时只需在外层网络边界精确放行业务、监控和运维来源；其中 `16060` 仅允许运维网段访问，不需要为 TDengine 单独启用本机 firewalld。

如果现场明确启用了 firewalld，把示例网段替换成现场网段后执行：

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.20.0/24" port port="16030" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.20.0/24" port port="16041" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.30.0/24" port port="16043" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.40.0/24" port port="16060" protocol="tcp" accept'
firewall-cmd --reload
firewall-cmd --list-rich-rules
```

统一申请并精确开放 `16030`、`16041`、`16043`、`16060`，禁止开放 `16030-16060` 整段端口。官方默认端口 `6030`、`6041`、`6043`、`6060` 不得开放。

---

## 13. 创建生产数据库

数据库名、保留天数和 vgroup 数必须来自业务容量评审。登录：

```bash
taos -h 127.0.0.1 -P 16030 -u root -p
```

示例：

```sql
CREATE DATABASE prod_iot
  KEEP 365d
  REPLICA 1
  VGROUPS 8;

SHOW DATABASES;
```

单体固定使用 `REPLICA 1`。`KEEP 365d` 和 `VGROUPS 8` 只是示例，上线前必须替换为容量评审结果。

---

## 14. 配置备份

### 14.1 执行一次全量备份

```bash
BACKUP_TIME="$(date +%F_%H%M%S)"

mkdir -p "/data/backup/tdengine/$BACKUP_TIME"
chmod 0750 "/data/backup/tdengine/$BACKUP_TIME"

taosdump \
  -h 127.0.0.1 \
  -P 16030 \
  -u root \
  -p \
  -A \
  -d snappy \
  -T 8 \
  -o "/data/backup/tdengine/$BACKUP_TIME"
```

生成校验文件：

```bash
find "/data/backup/tdengine/$BACKUP_TIME" -type f ! -name SHA256SUMS -printf '%P\0' \
  | sort -z \
  | (cd "/data/backup/tdengine/$BACKUP_TIME" && xargs -0 sha256sum) \
  > "/data/backup/tdengine/$BACKUP_TIME/SHA256SUMS"

du -sh "/data/backup/tdengine/$BACKUP_TIME"
(cd "/data/backup/tdengine/$BACKUP_TIME" && \
  sha256sum -c "/data/backup/tdengine/$BACKUP_TIME/SHA256SUMS")
```

备份完成后，必须把整个备份目录复制到另一台主机或独立备份存储。`SHA256SUMS` 使用相对路径，复制后仍可校验。

建议：每天至少一次全量备份；大库使用 `-S`、`-E` 按时间分段。OSS 不提供 Enterprise 的持续增量和时间点恢复能力。

### 14.2 做一次恢复演练

优先恢复到独立测试实例。下面以原库 `prod_iot`、恢复库 `restore_drill_20260713`、备份目录 `/data/backup/tdengine/2026-07-13_150000` 为例，执行前替换成实际值：

```bash
(cd /data/backup/tdengine/2026-07-13_150000 && \
  sha256sum -c /data/backup/tdengine/2026-07-13_150000/SHA256SUMS)

taosdump \
  -i /data/backup/tdengine/2026-07-13_150000 \
  -h 127.0.0.1 \
  -P 16030 \
  -u root \
  -p \
  -W "prod_iot=restore_drill_20260713"
```

登录后检查：

```sql
SHOW DATABASES;
USE <恢复库名>;
SHOW STABLES;
SHOW TABLES;
SELECT COUNT(*) FROM <代表性超级表>;
SELECT MIN(ts), MAX(ts) FROM <代表性超级表>;
```

`<恢复库名>` 填写新建的恢复演练库，不能填写生产库名。

数据量、时间范围、表结构和代表性查询全部正确，恢复演练才算通过。

---

## 15. 接入 Prometheus

Prometheus 增加：

```yaml
scrape_configs:
  - job_name: "tdengine-single"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["<TDengine主机IP>:16043"]
```

TDengine 3.4.1 总版本文档同时描述了 `/metrics/v2`，但本文目标包 `3.4.1.13` 实测该路径返回 `404`，因此单体部署固定使用 `/metrics`。

建议告警：

| 检查项 | 告警条件 |
|---|---|
| dnode 状态 | 非 ready 立即告警 |
| taosd/Adapter/Keeper | 服务停止立即告警 |
| CPU | 持续超过 50% 预警 |
| 数据盘 | 70% 警告，80% 严重 |
| 可用内存 | 持续不足或发生 OOM |
| TDengine 错误日志 | 持续增长 |
| 最近备份 | 超过既定 RPO 没有成功备份 |

本机检查：

```bash
curl -fsS http://127.0.0.1:16043/check_health
curl -fsS http://127.0.0.1:16043/metrics | sed -n '1,30p'
```

---

## 16. 升级和回滚

### 16.1 升级前必须完成

- 重新核验目标版本官方文档和发布说明；
- 校验并审计新 Generic 包；
- 确认客户端驱动兼容；
- 完成全量备份、异机复制和恢复演练；
- 保留旧版本安装包、解压目录和程序目录；
- 申请停机窗口。

### 16.2 升级步骤

下面用 `3.4.x.y` 表示经过验证的目标版本，执行前统一替换成实际版本号。

```bash
# 先停止业务写入
systemctl stop taos-explorer taoskeeper taosadapter taosd

# 解压已校验的新包后进入其目录
mkdir -p /data/module/tdengine/3.4.x.y/install
tar -zxf /data/module/tdengine/3.4.x.y/tdengine-tsdb-oss-3.4.x.y-linux-x64.tar.gz \
  -C /data/module/tdengine/3.4.x.y/install
cd /data/module/tdengine/3.4.x.y/install/tdengine-tsdb-oss-3.4.x.y
/data/module/tdengine/3.4.x.y/install/tdengine-tsdb-oss-3.4.x.y/install.sh \
  -v server -e no -d /data/module/tdengine/3.4.x.y

# 恢复生产安全基线并阻止安装器提前启动服务
systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
chmod 0750 /var/log/taos

# 检查新配置模板，人工合并需要的新增项
find /etc/taos -maxdepth 1 -type f -name '*.new' -ls

# 修正当前版本和数据日志软链接
ln -sfnT /data/module/tdengine/data /data/module/tdengine/3.4.x.y/taos/data
ln -sfnT /data/module/tdengine/logs/taosd /data/module/tdengine/3.4.x.y/taos/log
ln -sfnT /data/module/tdengine/3.4.x.y/taos /data/module/tdengine/current

# 确认 systemd 已指向新版本
readlink -f "$(command -v taosd)"
systemctl cat taosd | grep '^ExecStart='

systemctl start taosd
systemctl start taosadapter
systemctl start taoskeeper
systemctl start taos-explorer
taosd -V
```

服务启动后恢复业务写入。

### 16.3 回滚

只有官方确认可逆的同一 Maintenance 系列，才允许尝试原目录回滚：

```bash
systemctl stop taos-explorer taoskeeper taosadapter taosd

cd /data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13
/data/module/tdengine/3.4.1.13/install/tdengine-tsdb-oss-3.4.1.13/install.sh \
  -v server -e no -d /data/module/tdengine/3.4.1.13

systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
chmod 0750 /var/log/taos

ln -sfnT /data/module/tdengine/data /data/module/tdengine/3.4.1.13/taos/data
ln -sfnT /data/module/tdengine/logs/taosd /data/module/tdengine/3.4.1.13/taos/log
ln -sfnT /data/module/tdengine/3.4.1.13/taos /data/module/tdengine/current

systemctl start taosd
systemctl start taosadapter
systemctl start taoskeeper
systemctl start taos-explorer
taosd -V
```

跨 Feature/Major、元数据异常或原目录回滚失败时，停止反复启动，安装旧版本到干净环境并从升级前 `taosdump` 备份恢复。

不要在回滚观察期内删除旧版本目录，不要在生产使用 `uninstall.sh -e yes`。

---

## 17. 官方参考资料

以下资料于 2026-07-14 重新核验：

1. [TDengine Download Center](https://tdengine.com/downloads/)
2. [TDengine 3.4.1 Supported Platforms](https://docs.tdengine.com/3.4.1/reference/supported/)
3. [TDengine 3.4.1 System Requirements](https://docs.tdengine.com/3.4.1/operation/planning/)
4. [Deploy from Package](https://docs.tdengine.com/3.4.1/get-started/package/)
5. [taosd Reference](https://docs.tdengine.com/3.4.1/reference/components/taosd/)
6. [taosAdapter Reference](https://docs.tdengine.com/3.4.1/reference/components/taosadapter/)
7. [taosKeeper Reference](https://docs.tdengine.com/3.4.1/reference/components/taoskeeper/)
8. [taosExplorer Reference](https://docs.tdengine.com/3.4.1/reference/components/explorer/)
9. [Data Backup and Restoration](https://docs.tdengine.com/3.4.1/operation/backup/)
10. [taosdump Reference](https://docs.tdengine.com/3.4.1/reference/tools/taosdump/)
11. [Users and Permissions](https://docs.tdengine.com/3.4.1/operation/user/)
12. [TDengine 3.4.1.13 Release Notes](https://docs.tdengine.com/releases/notes/3.4.1.13/)
13. [TDengine Release History](https://docs.tdengine.com/3.4.1/releases/)
14. [Manual Deployment](https://docs.tdengine.com/3.4.1/operation/deployment/manual/)
15. [TDengine 3.4.1.13 Linux-Generic x64 官方包](https://downloads.taosdata.com/tdengine-tsdb-oss/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz)
16. [Java Connector](https://docs.tdengine.com/3.4.1/reference/connector/java/)

每次安装或升级仍需重新核验目标版本下载页、兼容矩阵、发布说明和备份恢复限制。
