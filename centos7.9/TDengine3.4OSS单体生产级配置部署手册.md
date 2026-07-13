# TDengine 3.4 OSS 单体生产级配置部署手册

> **Author:** 王有政<br>
> **日期:** 2026-07-13<br>
> **系统:** CentOS 7.9 x86_64<br>
> **版本:** TDengine TSDB-OSS 3.4.1.13<br>
> **范围:** 单体部署。集群、高可用和跨机房容灾不在本文展开。

---

## 使用说明

本文按实际部署顺序编写。第一次部署时，从第 1 节开始依次执行，不要跳过验证步骤。

本文已经固定以下选择，执行人员不需要再次做技术选型：

- 使用官方 Linux-Generic x64 压缩包；
- 程序安装到带版本目录 `/data/module/tdengine-3.4.1.13/taos`；
- 当前版本软链接为 `/data/module/tdengine`；
- 数据、日志和临时文件放在 `/data/tdengine`；
- 使用官方端口 `6030`、`6041`、`6043`；
- 启动 `taosd`、`taosadapter`、`taoskeeper`；
- 停用 `taos-explorer`；
- 使用 `taosdump` 做逻辑备份；
- 使用 taosKeeper `/metrics/v2` 接入 Prometheus。

执行约定：

1. 使用 root Shell 执行：`sudo -i`；
2. 文中的 `<...>` 必须替换成现场值；
3. 密码不得写入 Shell 命令、Git、工单正文或聊天记录；
4. 已经运行 TDengine 或存在历史数据时，不要使用本文“首次安装”步骤覆盖安装；
5. 集群部署不要套用本文。

---

## 1. 进入 root Shell 并设置部署变量

```bash
sudo -i

export TD_VERSION=3.4.1.13
export TD_PKG="tdengine-tsdb-oss-${TD_VERSION}-linux-x64.tar.gz"
export TD_URL="https://downloads.taosdata.com/tdengine-tsdb-oss/${TD_VERSION}/${TD_PKG}"
export TD_MD5=4c0fc6e31d59107a5125d5284fc58c32
export TD_SHA256=1b8cba22f1ea510ec72863041413ad86970ece2501d15c6eff482db189add515

export TD_SOFT_DIR="/data/soft/tdengine/${TD_VERSION}"
export TD_INSTALL_PARENT="/data/module/tdengine-${TD_VERSION}"
export TD_HOME="${TD_INSTALL_PARENT}/taos"
export TD_CURRENT=/data/module/tdengine

export TD_DATA=/data/tdengine/data
export TD_LOG=/data/tdengine/log
export TD_TMP=/data/tdengine/tmp
export TD_BACKUP=/backup/tdengine

export TD_HOST="$(hostnamectl --static)"
export TD_IP='<服务器内网IP>'
```

必须替换 `TD_IP`，然后检查变量：

```bash
printf 'TD_VERSION=%s\nTD_HOST=%s\nTD_IP=%s\nTD_HOME=%s\n' \
  "$TD_VERSION" "$TD_HOST" "$TD_IP" "$TD_HOME"

case "$TD_IP" in
  '<'*|'') echo 'ERROR: 请先填写 TD_IP'; exit 1 ;;
  127.*|0.*|169.254.*) echo 'ERROR: TD_IP 必须是服务器真实内网 IP'; exit 1 ;;
esac

```

如果中途退出 Shell，重新执行本节变量块后再继续。

---

## 2. 检查服务器

### 2.1 系统和资源

```bash
cat /etc/redhat-release
uname -m
hostnamectl
lscpu
free -h
lsblk -o NAME,SIZE,ROTA,TYPE,FSTYPE,MOUNTPOINT
df -hT
timedatectl
chronyc tracking
ulimit -n
```

继续部署前必须满足：

- CentOS 7.9、x86_64；
- 时间同步正常；
- `/data` 是已确认的数据盘；
- 推荐生产起步规格为 16C/64G；
- 数据盘按容量评审准备，并至少保留 30% 空闲空间；
- 已执行《CentOS7.9生产环境初始化手册.md》中的 Swap、THP、文件句柄和磁盘基线。

安装本文使用的基础命令；CentOS Yum 源应已由初始化手册配置完成：

```bash
yum install -y vim-enhanced wget openssl curl sysstat
```

### 2.2 检查历史 TDengine

```bash
rpm -qa | grep -Ei 'tdengine|taos' || true
command -v taos taosd taosadapter taoskeeper 2>/dev/null || true
systemctl list-unit-files | grep -Ei '^taos' || true
ps -ef | grep -E '[t]aosd|[t]aosadapter|[t]aoskeeper' || true
ss -lntup | grep -E ':(6030|6041|6043|6044|6045|6046|6047|6048|6049|6060)\b' || true
find /etc/taos /var/lib/taos /var/log/taos -maxdepth 2 -ls 2>/dev/null
```

预期结果是：没有已安装的 TDengine、没有相关进程、端口未占用、没有未知数据。

只要发现历史安装或数据，立即停止首次安装，先确认版本、数据目录、备份和业务状态。

---

## 3. 确认 hostname 和 FQDN

本文默认保留服务器现有静态 hostname，不无条件修改。

```bash
printf 'hostname=<%s>\n' "$(hostname)"
printf 'hostname-f=<%s>\n' "$(hostname -f)"
printf 'static=<%s>\n' "$(hostnamectl --static)"
getent ahostsv4 "$TD_HOST" || true
```

hostname 必须唯一、稳定，不包含空格和多余点号。

如果上面的 `static` 是空值、`localhost`、`localhost.localdomain`，或不符合企业命名规范，必须在首次安装前修改。已有合格 hostname 时不要改。

需要修改时执行，并把 `<新hostname>` 换成正式名称：

```bash
hostnamectl set-hostname <新hostname>
export TD_HOST="$(hostnamectl --static)"
printf 'TD_HOST=%s\n' "$TD_HOST"
```

确认最终 hostname 合格：

```bash
case "$TD_HOST" in
  ''|localhost|localhost.localdomain|*' '*|*..*|.*|*.)
    echo "ERROR: hostname 仍不合格：$TD_HOST"
    exit 1
    ;;
esac
```

如果 `getent` 没有返回 `$TD_IP`，先备份再编辑 `/etc/hosts`：

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
getent ahostsv4 "$TD_HOST"
getent hosts "$TD_HOST"
getent ahostsv4 "$TD_HOST" | awk '{print $1}' | grep -Fx "$TD_IP"

if grep -E '^(127\.[0-9.]+|::1)[[:space:]]' /etc/hosts | grep -Fw "$TD_HOST"; then
  echo "ERROR: $TD_HOST 仍然映射到了回环地址，请修正 /etc/hosts"
  exit 1
fi

# 现场允许 ICMP 时执行
ping -c 3 "$TD_HOST"
```

只有 `getent` 返回真实内网 IP，并且回环地址冲突检查没有输出，才能继续部署。

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
install -d -m 0755 -o root -g root \
  /data/soft \
  /data/module \
  "$TD_SOFT_DIR" \
  "$TD_INSTALL_PARENT"

install -d -m 0750 -o root -g root \
  "$TD_DATA" \
  "$TD_LOG/taosd" \
  "$TD_LOG/adapter" \
  "$TD_LOG/keeper" \
  "$TD_TMP" \
  "$TD_BACKUP"

df -hT /data /backup 2>/dev/null || true
namei -om "$TD_DATA"
```

目录用途：

| 路径 | 用途 |
|---|---|
| `/data/soft/tdengine/3.4.1.13` | 安装包和解压目录 |
| `/data/module/tdengine-3.4.1.13/taos` | 程序版本目录 |
| `/data/module/tdengine` | 当前版本软链接 |
| `/data/tdengine/data` | 数据和 WAL |
| `/data/tdengine/log/*` | 各组件日志 |
| `/data/tdengine/tmp` | 临时文件 |
| `/backup/tdengine` | 本机备份暂存 |

`/backup` 必须位于独立存储或将备份继续复制到其他主机；与数据同盘不算有效灾备。

---

## 6. 获取并校验安装包

进入制品目录：

```bash
cd "$TD_SOFT_DIR"
```

生产环境推荐从企业制品库上传到该目录。如果当前服务器允许访问官方 CDN，可以执行：

```bash
wget -O "$TD_PKG" "$TD_URL"
```

校验文件：

```bash
test "$(stat -c '%s' "$TD_PKG")" -eq 300274656
echo "${TD_MD5}  ${TD_PKG}" | md5sum -c -
echo "${TD_SHA256}  ${TD_PKG}" | sha256sum -c -
gzip -t "$TD_PKG"
```

预期三项均成功：

```text
MD5: OK
SHA256: OK
gzip: 无错误输出
```

`TD_SHA256` 是 2026-07-13 对官方 CDN 文件的实算值，不是厂商数字签名。企业制品库仍应保存自己的 SHA256 清单和下载审计记录。

---

## 7. 解压并安装

### 7.1 解压

```bash
cd "$TD_SOFT_DIR"

test ! -e "tdengine-tsdb-oss-${TD_VERSION}" || {
  echo 'ERROR: 解压目录已存在，请确认是否为上次残留'
  exit 1
}

tar -tzf "$TD_PKG" | head -30
tar -zxf "$TD_PKG"
cd "tdengine-tsdb-oss-${TD_VERSION}"
```

确认安装脚本参数：

```bash
./install.sh -h
sha256sum install.sh start-all.sh stop-all.sh package.tar.gz
```

### 7.2 执行安装

```bash
./install.sh \
  -v server \
  -e no \
  -d "$TD_INSTALL_PARENT"
```

注意：`-d` 指定的是父目录，安装器会自动追加 `/taos`。最终目录应为：

```text
/data/module/tdengine-3.4.1.13/taos
```

### 7.3 安装后立即收敛

安装器会启用 Explorer，并可能把 `/var/log/taos` 权限设置为 `777`，立即处理：

```bash
systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
systemctl disable taos-explorer
chmod 0750 /var/log/taos
```

创建当前版本软链接并修正安装器的 data/log 软链接：

```bash
[ ! -e "$TD_CURRENT" ] || [ -L "$TD_CURRENT" ] || {
  echo "ERROR: $TD_CURRENT 不是软链接"
  exit 1
}

test -L "$TD_HOME/data"
test -L "$TD_HOME/log"

ln -sfn "$TD_HOME" "$TD_CURRENT"
ln -sfn "$TD_DATA" "$TD_HOME/data"
ln -sfn "$TD_LOG/taosd" "$TD_HOME/log"
```

### 7.4 验证安装结果

```bash
test -x "$TD_HOME/bin/taosd"
cat "$TD_HOME/.install_path"

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
```

预期：

- `taosd` 实际路径位于 `$TD_HOME/bin`；
- `taosd`、`taosadapter`、`taoskeeper` 已启用但尚未启动；
- `taos-explorer` 为 disabled；
- systemd 的 `ExecStart` 指向带版本目录；
- `/usr/bin` 中的 TDengine 命令是软链接。

### 7.5 配置环境变量

和 MySQL、Redis、JDK 手册保持一致，TDengine 使用当前版本软链接配置统一环境变量。

编辑团队环境变量文件；如果已经存在其他软件变量，只追加 TDengine 内容，不得覆盖整个文件：

```bash
vim /etc/profile.d/myenv.sh
```

追加：

```bash
# TDENGINE_HOME：TDengine 当前版本软链接，升级时只调整软链接
export TDENGINE_HOME=/data/module/tdengine
export PATH=$TDENGINE_HOME/bin:$PATH
```

保存并加载：

```bash
chown root:root /etc/profile.d/myenv.sh
chmod 0644 /etc/profile.d/myenv.sh
source /etc/profile.d/myenv.sh

echo "$TDENGINE_HOME"
which taos taosd taosdump
readlink -f "$(command -v taosd)"
taos -V
taosd -V
```

预期 `which` 指向 `/data/module/tdengine/bin/`，`readlink -f` 最终指向当前带版本目录。

systemd 使用绝对 `ExecStart`，不依赖交互式 Shell 环境变量。

---

## 8. 配置 taosd

备份配置：

```bash
cp -a /etc/taos/taos.cfg "/etc/taos/taos.cfg.$(date +%F_%H%M%S).bak"
vim /etc/taos/taos.cfg
```

在文件中查找并保证以下每个参数只有一个有效值。把 `<当前hostname>` 替换为 `echo "$TD_HOST"` 的输出：

```ini
firstEp             <当前hostname>:6030
fqdn                <当前hostname>
serverPort          6030

dataDir             /data/tdengine/data
logDir              /data/tdengine/log/taosd
tempDir             /data/tdengine/tmp

minimalDataDirGB    20
minimalLogDirGB     5
minimalTmpDirGB     5

asyncLog            1
numOfLogLines       10000000

monitor             1
monitorFqdn         <当前hostname>
monitorPort         6043
monitorInterval     30
```

安装脚本可能已经写入 `monitorFQDN`。必须删除重复项，统一保留一个 `monitorFqdn`。

检查配置：

```bash
grep -niE '^(firstEp|fqdn|serverPort|dataDir|logDir|tempDir|minimalDataDirGB|minimalLogDirGB|minimalTmpDirGB|asyncLog|numOfLogLines|monitor|monitorFqdn|monitorPort|monitorInterval)[[:space:]]' \
  /etc/taos/taos.cfg
```

确认输出没有重复参数，并且 hostname、端口和目录正确。

---

## 9. 配置 taosAdapter

备份并编辑：

```bash
cp -a /etc/taos/taosadapter.toml "/etc/taos/taosadapter.toml.$(date +%F_%H%M%S).bak"
vim /etc/taos/taosadapter.toml
```

确认或修改以下已有参数：

```toml
debug = false
port = 6041
taosConfigDir = "/etc/taos"

[log]
path = "/data/tdengine/log/adapter"
level = "info"
keepDays = 30
rotationCount = 30
rotationSize = "1GB"
compress = true
reservedDiskSize = "5GB"

[uploadKeeper]
enable = true
url = "http://127.0.0.1:6043/adapter_report"
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
```

不要追加包内不存在的配置段。只修改实际文件中已有参数。

---

## 10. 启动 taosd 和 taosAdapter

```bash
systemctl daemon-reload
systemctl enable --now taosd
systemctl enable --now taosadapter

systemctl status taosd --no-pager -l
systemctl status taosadapter --no-pager -l
ss -lntp | grep -E ':(6030|6041)\b'
journalctl -u taosd -u taosadapter --since '-10 min' --no-pager
```

必须看到：

- `taosd` 和 `taosadapter` 为 `active (running)`；
- `6030/TCP`、`6041/TCP` 正在监听；
- 日志没有持续报错。

不要执行 `start-all.sh`。3.4.1.13 的该脚本会同时启动 Explorer。

---

## 11. 修改 root 密码并启动 taosKeeper

### 11.1 生成密码

```bash
TD_ROOT_PASS="TdE!9$(openssl rand -hex 12)"
printf '请立即保存到企业密码库：%s\n' "$TD_ROOT_PASS"
```

该命令生成至少包含大写、小写、数字和特殊字符的随机密码。不要把输出复制到工单或聊天记录。

### 11.2 修改密码

```bash
taos -h 127.0.0.1 -P 6030 -u root -p
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
taos -h 127.0.0.1 -P 6030 -u root -p
```

### 11.3 配置 taosKeeper

```bash
cp -a /etc/taos/taoskeeper.toml "/etc/taos/taoskeeper.toml.$(date +%F_%H%M%S).bak"
vim /etc/taos/taoskeeper.toml
```

配置：

```toml
instanceId = 64
host = "127.0.0.1"
port = 6043
RotationInterval = "15s"

[tdengine]
host = "127.0.0.1"
port = 6041
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

[prometheus]
includeTables = []
cacheTTL = 300

[log]
path = "/data/tdengine/log/keeper"
level = "info"
rotationCount = 7
keepDays = 30
rotationSize = "1GB"
compress = true
reservedDiskSize = "5GB"
```

限制配置文件权限：

```bash
chown root:root /etc/taos/taoskeeper.toml
chmod 0600 /etc/taos/taoskeeper.toml
unset TD_ROOT_PASS
```

启动并验证：

```bash
systemctl enable --now taoskeeper
systemctl status taoskeeper --no-pager -l
curl -fsS http://127.0.0.1:6043/check_health
curl -fsS 'http://127.0.0.1:6043/metrics/v2?max_age=30' | head
```

---

## 12. 配置端口和防火墙

### 12.1 本文使用的端口

| 端口 | 用途 | 开放范围 |
|---:|---|---|
| `6030/TCP` | Native 客户端、taosd | 仅 Native 客户端和运维网，按需开放 |
| `6041/TCP` | REST/WebSocket、taosAdapter | 业务应用网段 |
| `6043/TCP` | taosKeeper、Prometheus | 监控网段 |
| `6044～6049` | 可选采集协议 | 不开放 |
| `6060/TCP` | Explorer | 不开放，服务保持 disabled |

端口已经登记到《服务器端口统一规划规范.md》。数据库端口不得对公网开放。

### 12.2 检查 firewalld

```bash
systemctl is-active firewalld 2>/dev/null || true
systemctl is-enabled firewalld 2>/dev/null || true
```

仓库服务器初始化基线默认由安全组、硬件防火墙或统一 ACL 控制，firewalld 通常为关闭状态。此时只需在外层网络边界精确放行业务和监控来源。

如果现场明确启用了 firewalld，填写来源网段后执行：

```bash
export APP_CIDR='<业务应用网段，例如10.10.20.0/24>'
export MONITOR_CIDR='<Prometheus网段，例如10.10.30.0/24>'

case "${APP_CIDR}${MONITOR_CIDR}" in
  *'<'*) echo 'ERROR: 请先填写 APP_CIDR 和 MONITOR_CIDR'; exit 1 ;;
esac

firewall-cmd --permanent --add-rich-rule="rule family=\"ipv4\" source address=\"${APP_CIDR}\" port port=\"6041\" protocol=\"tcp\" accept"
firewall-cmd --permanent --add-rich-rule="rule family=\"ipv4\" source address=\"${MONITOR_CIDR}\" port port=\"6043\" protocol=\"tcp\" accept"
firewall-cmd --reload
firewall-cmd --list-rich-rules
```

业务不使用 Native 驱动时，不开放 `6030`。禁止开放 `6030-6060` 整段端口。

---

## 13. 创建生产数据库

数据库名、保留天数和 vgroup 数必须来自业务容量评审。登录：

```bash
taos -h 127.0.0.1 -P 6030 -u root -p
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
BACKUP_ID="$(date +%F_%H%M%S)"
BACKUP_DIR="${TD_BACKUP}/${BACKUP_ID}"

install -d -m 0750 -o root -g root "$BACKUP_DIR"

taosdump \
  -h 127.0.0.1 \
  -P 6030 \
  -u root \
  -p \
  -A \
  -d snappy \
  -T 8 \
  -o "$BACKUP_DIR"
```

生成校验文件：

```bash
(cd "$BACKUP_DIR" && \
  find . -type f -print0 \
    | sort -z \
    | xargs -0 sha256sum > SHA256SUMS)

du -sh "$BACKUP_DIR"
(cd "$BACKUP_DIR" && sha256sum -c SHA256SUMS)
```

备份完成后，必须把整个备份目录复制到另一台主机或独立备份存储。`SHA256SUMS` 使用相对路径，复制后仍可校验。

建议：每天至少一次全量备份；大库使用 `-S`、`-E` 按时间分段。OSS 不提供 Enterprise 的持续增量和时间点恢复能力。

### 14.2 做一次恢复演练

优先恢复到独立测试实例。只能使用当前实例演练时，必须把备份内容恢复成新的隔离库，绝不能覆盖生产库：

```bash
export SOURCE_DB='<原数据库名>'
export RESTORE_DB="restore_drill_$(date +%Y%m%d)"
export RESTORE_DIR='<备份目录>'

case "${SOURCE_DB}${RESTORE_DIR}" in
  *'<'*) echo 'ERROR: 请先填写 SOURCE_DB 和 RESTORE_DIR'; exit 1 ;;
esac

printf '原库=%s\n恢复演练库=%s\n备份目录=%s\n' \
  "$SOURCE_DB" "$RESTORE_DB" "$RESTORE_DIR"

(cd "$RESTORE_DIR" && sha256sum -c SHA256SUMS)

taosdump \
  -i "$RESTORE_DIR" \
  -h 127.0.0.1 \
  -P 6030 \
  -u root \
  -p \
  -W "${SOURCE_DB}=${RESTORE_DB}"
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

`<恢复库名>` 使用上一步输出的“恢复演练库”，不要填写生产库名。

数据量、时间范围、表结构和代表性查询全部正确，恢复演练才算通过。

---

## 15. 接入 Prometheus

Prometheus 增加：

```yaml
scrape_configs:
  - job_name: "tdengine-single"
    metrics_path: "/metrics/v2"
    params:
      max_age: ["30"]
    static_configs:
      - targets: ["<TDengine主机IP>:6043"]
```

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
curl -fsS http://127.0.0.1:6043/check_health
curl -fsS 'http://127.0.0.1:6043/metrics/v2?max_age=30' | head
```

---

## 16. 上线前验证清单

### 16.1 安装和配置

- [ ] CentOS 7.9 x86_64；
- [ ] hostname 唯一并解析到内网 IP；
- [ ] 官方包 MD5、SHA256、gzip 校验通过；
- [ ] 程序位于 `/data/module/tdengine-3.4.1.13/taos`；
- [ ] `/data/module/tdengine` 指向当前版本；
- [ ] data/log/tmp 指向 `/data/tdengine`；
- [ ] `taos-explorer` 为 disabled；
- [ ] `/var/log/taos` 不是 `777`；
- [ ] `taos.cfg` 没有重复参数；
- [ ] root 默认密码已经修改；
- [ ] taosKeeper 配置权限为 `0600`。

### 16.2 服务和端口

```bash
systemctl is-active taosd taosadapter taoskeeper
systemctl is-enabled taosd taosadapter taoskeeper
systemctl is-enabled taos-explorer || true
ss -lntup | grep -E ':(6030|6041|6043|6044|6045|6046|6047|6048|6049|6060)\b'
```

- [ ] taosd、taosadapter、taoskeeper 为 active/enabled；
- [ ] 仅监听审批通过的 `6030`、`6041`、`6043`；
- [ ] `6044～6049`、`6060` 未对外开放；
- [ ] 数据库端口未对公网开放。

### 16.3 读写验证

```bash
taos -h 127.0.0.1 -P 6030 -u root -p
```

```sql
CREATE DATABASE verify_tdengine KEEP 7d REPLICA 1 VGROUPS 2;
USE verify_tdengine;

CREATE STABLE meters (
  ts TIMESTAMP,
  current FLOAT,
  voltage INT
) TAGS (
  location VARCHAR(64),
  group_id INT
);

CREATE TABLE d001 USING meters TAGS ('beijing', 1);
INSERT INTO d001 VALUES (NOW, 10.2, 220);
SELECT * FROM meters;
FLUSH DATABASE verify_tdengine;
DROP DATABASE verify_tdengine;
```

- [ ] Native CLI 读写成功；
- [ ] 业务实际使用的 WebSocket/REST 连接成功；
- [ ] 应用断开重连成功；
- [ ] 典型业务查询结果正确。

### 16.4 备份和监控

- [ ] 全量备份成功；
- [ ] SHA256 校验成功；
- [ ] 备份已复制到独立存储；
- [ ] 恢复演练成功；
- [ ] `/check_health` 成功；
- [ ] Prometheus 抓取成功；
- [ ] 告警已经测试。

### 16.5 重启验证

在维护窗口执行：

```bash
systemctl restart taosd
systemctl restart taosadapter
systemctl restart taoskeeper
systemctl is-active taosd taosadapter taoskeeper
```

条件允许时完成一次操作系统重启，确认数据盘挂载、服务、端口、数据和监控全部恢复。

---

## 17. 日常运维

```bash
# 服务
systemctl status taosd taosadapter taoskeeper --no-pager
journalctl -u taosd --since '-30 min' --no-pager
journalctl -u taosadapter --since '-30 min' --no-pager
journalctl -u taoskeeper --since '-30 min' --no-pager

# 资源
df -hT /data /backup 2>/dev/null
du -sh /data/tdengine/data /data/tdengine/log/* /data/tdengine/tmp
iostat -x 1 5
chronyc tracking

# TDengine
taos -h 127.0.0.1 -P 6030 -u root -p
```

常用 SQL：

```sql
SHOW DNODES;
SHOW DATABASES;
SELECT * FROM information_schema.ins_dnodes;
SELECT * FROM information_schema.ins_vnodes;
USE log;
SHOW STABLES;
```

停止顺序：先停止业务写入，再停止 Keeper、Adapter、taosd。

启动顺序：taosd、Adapter、Keeper，验证通过后恢复业务写入。

---

## 18. 升级和回滚

### 18.1 升级前必须完成

- 重新核验目标版本官方文档和发布说明；
- 校验并审计新 Generic 包；
- 确认客户端驱动兼容；
- 完成全量备份、异机复制和恢复演练；
- 保留旧版本安装包、解压目录和程序目录；
- 申请停机窗口。

### 18.2 升级步骤

```bash
export OLD_VERSION=3.4.1.13
export NEW_VERSION='<目标版本>'
export NEW_PARENT="/data/module/tdengine-${NEW_VERSION}"
export NEW_HOME="${NEW_PARENT}/taos"

case "$NEW_VERSION" in
  '<'*|'') echo 'ERROR: 请先填写 NEW_VERSION'; exit 1 ;;
esac

# 先停止业务写入
systemctl stop taoskeeper taosadapter taosd

# 解压已校验的新包后进入其目录
cd "/data/soft/tdengine/${NEW_VERSION}/tdengine-tsdb-oss-${NEW_VERSION}"
install -d -m 0755 -o root -g root "$NEW_PARENT"
./install.sh -v server -e no -d "$NEW_PARENT"

# 恢复生产安全基线
systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
systemctl disable taos-explorer
chmod 0750 /var/log/taos

# 检查新配置模板，人工合并需要的新增项
find /etc/taos -maxdepth 1 -type f -name '*.new' -ls

# 修正当前版本和数据日志软链接
test -L "$NEW_HOME/data"
test -L "$NEW_HOME/log"
ln -sfn "$NEW_HOME" /data/module/tdengine
ln -sfn /data/tdengine/data "$NEW_HOME/data"
ln -sfn /data/tdengine/log/taosd "$NEW_HOME/log"

# 确认 systemd 已指向新版本
readlink -f "$(command -v taosd)"
systemctl cat taosd | grep '^ExecStart='

systemctl start taosd
systemctl start taosadapter
systemctl start taoskeeper
taosd -V
```

执行第 16 节验证，通过后恢复业务写入。

### 18.3 回滚

只有官方确认可逆的同一 Maintenance 系列，才允许尝试原目录回滚：

```bash
systemctl stop taoskeeper taosadapter taosd

cd "/data/soft/tdengine/${OLD_VERSION}/tdengine-tsdb-oss-${OLD_VERSION}"
./install.sh -v server -e no -d "/data/module/tdengine-${OLD_VERSION}"

systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
systemctl disable taos-explorer
chmod 0750 /var/log/taos

OLD_HOME="/data/module/tdengine-${OLD_VERSION}/taos"
ln -sfn "$OLD_HOME" /data/module/tdengine
ln -sfn /data/tdengine/data "$OLD_HOME/data"
ln -sfn /data/tdengine/log/taosd "$OLD_HOME/log"

systemctl start taosd
systemctl start taosadapter
systemctl start taoskeeper
taosd -V
```

跨 Feature/Major、元数据异常或原目录回滚失败时，停止反复启动，安装旧版本到干净环境并从升级前 `taosdump` 备份恢复。

不要在回滚观察期内删除旧版本目录，不要在生产使用 `uninstall.sh -e yes`。

---

## 19. 安装方案和 RPM 校验说明

部署方案比较如下。主流程已经选择方案 A，现场无需再次选择。

| 方案 | 优点 | 缺点 | 结论 |
|---|---|---|---|
| A：Generic + 自定义版本目录 | 版本目录清楚，脚本可审计，便于保留旧版本 | 仍会修改系统级链接和 unit | **本文采用** |
| B：RPM + 本地 Yum | 包管理和依赖记录最好 | 程序路径由 RPM 定义 | 大规模包管理环境可选 |
| C：RPM 直接安装 | 命令简单 | 依赖需手工处理 | 不推荐作为团队主流程 |
| D：Docker | 环境封装 | 数据卷、网络和升级链路更复杂 | 已有成熟容器平台时使用 |

如果改用 RPM，必须校验：

```bash
RPM_FILE=tdengine-tsdb-oss-3.4.1.13-linux-x64.rpm
RPM_MD5=e1ad92cbdcc3d459243e3311e7b259e4

echo "${RPM_MD5}  ${RPM_FILE}" | md5sum -c -
sha256sum "$RPM_FILE"
rpm -Kv "$RPM_FILE"
rpm -qp --qf 'NAME=%{NAME}\nVERSION=%{VERSION}\nRELEASE=%{RELEASE}\nARCH=%{ARCH}\n' "$RPM_FILE"
rpm -qpR "$RPM_FILE"
rpm -qlp "$RPM_FILE" | less
```

禁止使用 `rpm --nodeps` 强行安装。

---

## 20. 风险说明

| 风险 | 控制措施 |
|---|---|
| 单体是单点，主机故障即服务中断 | 明确 SLA、异机备份；需要高可用时编制集群方案 |
| CentOS 7 已 EOL | 严格网络隔离并制定操作系统迁移计划 |
| 3.4.1 未被官网标注为 LTS | 预生产验证、压测、恢复演练；后续评估 LTS/新维护版本 |
| 3.4.1.13 发布说明链接当前失效 | 上线前向官方或社区补齐变更和已知缺陷信息 |
| Generic 安装器修改系统路径 | 保留安装前后记录，使用带版本目录，升级后检查全局链接和 unit |
| CentOS 7 下服务默认以 root 运行 | 最小端口、网络白名单、严格配置权限、停用 Explorer |
| Keeper 配置保存数据库密码 | 文件权限 `0600`，密码只存企业密码库 |
| OSS 没有持续增量和 PITR | 全量/分段备份、异机复制、定期恢复演练 |
| 磁盘写满 | 70%/80% 告警、预留 30%、提前扩容 |
| REST 返回大结果集可能占用大量内存 | 应用限制时间范围和返回量，优先使用 WebSocket 流式读取 |
| 修改 hostname/fqdn 影响节点身份 | 初始化前固化，运行后变更必须制定迁移方案 |

生产评审必须接受：备份能够改善数据恢复，但不能让单体实例具备高可用能力。

---

## 21. 官方参考资料

以下资料于 2026-07-13 重新核验：

1. [TDengine Download Center](https://tdengine.com/downloads/)
2. [TDengine 3.4.1 Supported Platforms](https://docs.tdengine.com/3.4.1/reference/supported/)
3. [TDengine 3.4.1 System Requirements](https://docs.tdengine.com/3.4.1/operation/planning/)
4. [Deploy from Package](https://docs.tdengine.com/3.4.1/get-started/package/)
5. [taosd Reference](https://docs.tdengine.com/3.4.1/reference/components/taosd/)
6. [taosAdapter Reference](https://docs.tdengine.com/3.4.1/reference/components/taosadapter/)
7. [taosKeeper Reference](https://docs.tdengine.com/3.4.1/reference/components/taoskeeper/)
8. [Data Backup and Restoration](https://docs.tdengine.com/3.4.1/operation/backup/)
9. [taosdump Reference](https://docs.tdengine.com/3.4.1/reference/tools/taosdump/)
10. [Users and Permissions](https://docs.tdengine.com/3.4.1/operation/user/)
11. [TDengine Release History](https://docs.tdengine.com/3.4.1/releases/)
12. [Manual Deployment](https://docs.tdengine.com/3.4.1/operation/deployment/manual/)
13. [TDengine 3.4.1.13 Linux-Generic x64 官方包](https://downloads.taosdata.com/tdengine-tsdb-oss/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz)

每次安装或升级仍需重新核验目标版本下载页、兼容矩阵、发布说明和备份恢复限制。
