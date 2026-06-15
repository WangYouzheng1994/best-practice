# CentOS 7.9 内网集群服务器增量配置手册

> **Author:** 王有政  
> **日期:** 2026-06-15  
> **适用系统:** CentOS 7.9 (x86_64)  
> **文档定位:** 本文是 `CentOS7.9生产环境初始化手册.md` 的增量文档，用于指导已经完成单台服务器初始化的 CentOS 7.9 节点，进一步接入生产内网集群。  
> **重要说明:** 本文允许覆盖或补充初始化手册中的部分配置。单机初始化关注“单台可用与安全”，集群增量关注“多节点一致、互信、可观测、可维护、可扩展”。

---

## 目录

### 规划与准入

1. [适用范围与集群原则](#1-适用范围与集群原则)
2. [集群角色与网络规划](#2-集群角色与网络规划)
3. [节点接入前检查](#3-节点接入前检查)

### 基础一致性（P0）

1. [主机命名与资产标识](#4-主机命名与资产标识)
2. [内网 DNS 与 hosts 策略](#5-内网-dns-与-hosts-策略)
3. [集群时间同步体系](#6-集群时间同步体系)
4. [Yum 内网仓库与软件版本基线](#7-yum-内网仓库与软件版本基线)
5. [系统参数基线一致性](#8-系统参数基线一致性)
6. [用户、sudo 与 SSH 运维通道](#9-用户sudo-与-ssh-运维通道)

### 网络与安全（P0）

1. [网络分区与路由规范](#10-网络分区与路由规范)
2. [防火墙白名单与端口矩阵](#11-防火墙白名单与端口矩阵)
3. [节点间安全互信策略](#12-节点间安全互信策略)
4. [证书与内部 CA 管理](#13-证书与内部-ca-管理)
5. [内网访问边界与跳板机](#14-内网访问边界与跳板机)

### 存储、日志与可观测（P1）

1. [数据盘与目录规范](#15-数据盘与目录规范)
2. [集中日志采集](#16-集中日志采集)
3. [监控 Agent 与告警基线](#17-监控-agent-与告警基线)
4. [审计日志集中化](#18-审计日志集中化)
5. [备份、快照与恢复演练](#19-备份快照与恢复演练)

### 运维治理（P1）

1. [配置管理与批量变更](#20-配置管理与批量变更)
2. [服务发现与负载均衡接入](#21-服务发现与负载均衡接入)
3. [变更窗口与回滚策略](#22-变更窗口与回滚策略)
4. [故障域与高可用部署规范](#23-故障域与高可用部署规范)
5. [容量规划与压测准入](#24-容量规划与压测准入)
6. [最终验收清单](#25-最终验收清单)

---

## 1. 适用范围与集群原则

**目的：** 明确本文解决的问题，避免把单机初始化、业务组件部署和集群治理混在一起。

本文适用于以下场景：

- 已按 `CentOS7.9生产环境初始化手册.md` 完成基础初始化；
- 多台服务器需要组成一个生产内网集群；
- 集群内存在数据库、Redis、Nacos、应用、网关、日志、监控、备份等多类角色；
- 节点之间需要统一时间、名称解析、软件源、系统参数、安全策略、日志和监控；
- 服务器默认不直接暴露公网，通过跳板机、VPN、堡垒机或专线管理。

集群建设核心原则：

1. **先规划，后接入。** IP、主机名、角色、端口、故障域、数据目录必须先登记再实施。
2. **所有节点必须时间一致。** 时间漂移会影响日志排序、证书校验、分布式锁、Token、数据库复制和故障切换。
3. **配置基线应可重复。** 禁止只在单台机器手工改配置而不记录，所有集群级配置必须能复现。
4. **内网不等于安全。** 内网集群仍必须做最小权限、端口白名单、审计、密钥轮换和访问边界。
5. **按角色开放端口。** 不允许所有节点互相全端口放通，除非是临时排障且必须有过期时间。
6. **故障域隔离。** 主从、副本、仲裁、网关实例不得集中在同一物理机、同一机柜或同一宿主机。
7. **可观测先于上线。** 未接入监控、日志、告警和备份的节点，不应承载生产流量。
8. **配置冲突以集群文档为准。** 本文是增量文档，若与单机初始化手册存在冲突，应按本集群文档重新评估并记录原因。

---

## 2. 集群角色与网络规划

**目的：** 在执行配置前确定集群拓扑，避免后续出现 IP 冲突、端口冲突、故障域集中和命名混乱。

### 2.1 推荐网络分区


| 网络区域  | 示例网段            | 用途                   | 访问原则           |
| ----- | --------------- | -------------------- | -------------- |
| 管理网   | `10.10.10.0/24` | SSH、堡垒机、配置管理、监控采集    | 仅运维入口可访问       |
| 业务网   | `10.10.20.0/24` | 应用节点之间 RPC、HTTP、gRPC | 按服务端口白名单访问     |
| 数据网   | `10.10.30.0/24` | MySQL、Redis、MQ、存储复制  | 仅业务节点和同类节点访问   |
| 监控日志网 | `10.10.40.0/24` | Prometheus、日志采集、审计上报 | Agent 到服务端单向优先 |
| 备份网   | `10.10.50.0/24` | 备份、归档、快照传输           | 仅备份服务器与被备份节点互通 |


> **推荐：** 至少区分管理网、业务网、数据网。条件不足时也要通过防火墙规则实现逻辑隔离。

### 2.2 推荐节点角色表


| 角色       | 主机名示例                             | 数量建议 | 说明                                    |
| -------- | --------------------------------- | ---- | ------------------------------------- |
| 跳板机/堡垒机  | `prod-mom-bastion01`              | 1-2  | 运维统一入口，不承载业务                          |
| 时间服务器    | `prod-mom-ntp01`、`prod-mom-ntp02` | 2    | 集群内部时间源，至少 1 主 1 备                    |
| DNS/内网解析 | `prod-mom-dns01`、`prod-mom-dns02` | 2    | 内网域名解析，避免大规模 hosts 漂移                 |
| 应用节点     | `prod-mom-app01`~`03`             | >= 2 | 承载业务应用                                |
| 网关节点     | `prod-mom-gw01`~`02`              | >= 2 | Nginx、OpenResty、API 网关等               |
| 数据库节点    | `prod-mom-db01`~`03`              | 按架构  | MySQL 主从、MGR、备库等                      |
| 缓存节点     | `prod-mom-redis01`~`06`           | 按架构  | Redis 单体、主从或 Cluster                  |
| 注册配置中心   | `prod-mom-nacos01`~`03`           | >= 3 | Nacos、Consul、ZooKeeper 等              |
| 监控节点     | `prod-mom-mon01`~`02`             | >= 1 | Prometheus、Grafana、Alertmanager       |
| 日志节点     | `prod-mom-log01`~`03`             | 按容量  | Elasticsearch、OpenSearch、Loki、Kafka 等 |
| 备份节点     | `prod-mom-backup01`               | >= 1 | 备份存储与恢复验证                             |


### 2.3 主机规划登记模板

上线前先登记以下信息：


| 项目         | 示例                             | 是否必填 |
| ---------- | ------------------------------ | ---- |
| 环境         | `prod` / `test` / `dev`        | 是    |
| 业务线        | `mom`                          | 是    |
| 角色         | `app` / `db` / `redis` / `ntp` | 是    |
| 序号         | `01`                           | 是    |
| 主机名        | `prod-mom-app01`               | 是    |
| 管理 IP      | `10.10.10.21`                  | 是    |
| 业务 IP      | `10.10.20.21`                  | 视角色  |
| 数据 IP      | `10.10.30.21`                  | 视角色  |
| 机柜/宿主机/可用区 | `rack-a`                       | 是    |
| 系统盘        | `100G SSD`                     | 是    |
| 数据盘        | `/data 500G SSD`               | 视角色  |
| 负责人        | `运维/业务 owner`                  | 是    |


---

## 3. 节点接入前检查

**目的：** 确认单台服务器已经具备加入集群的最低条件。

```bash
# 系统版本
cat /etc/redhat-release
uname -r

# 主机名和 IP
hostnamectl status
ip addr show
ip route

# 时间状态
timedatectl status
chronyc tracking 2>/dev/null || true

# DNS 与 hosts
cat /etc/resolv.conf
cat /etc/hosts

# 磁盘与挂载
df -hT
lsblk
mount | column -t

# 防火墙状态
systemctl status firewalld --no-pager
firewall-cmd --state 2>/dev/null || true
firewall-cmd --list-all 2>/dev/null || true

# SSH 配置
sshd -T | egrep 'port|permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|allowgroups'

# 系统参数基线
sysctl net.ipv4.ip_forward net.ipv4.tcp_tw_reuse net.core.somaxconn vm.swappiness fs.file-max

# 当前运行服务
systemctl list-units --type=service --state=running
```

**准入标准：**


| 项目   | 要求                              |
| ---- | ------------------------------- |
| 系统版本 | CentOS 7.9 x86_64               |
| 主机名  | 符合集群命名规范且唯一                     |
| IP   | 无冲突，路由正确，网关可达                   |
| 时间   | 已配置 chrony，时间偏差小于 100ms         |
| SSH  | 仅允许受控入口访问，禁止 root 直接登录          |
| 防火墙  | 已启用或有等效安全组策略                    |
| 监控   | 至少预留 Node Exporter 或监控 Agent 端口 |
| 数据盘  | 业务数据不得直接写入系统盘                   |


---

## 4. 主机命名与资产标识

**目的：** 统一主机身份，避免日志、监控、告警和自动化脚本中出现不可识别节点。

### 4.1 命名规范

推荐格式：

```text
{环境}-{业务}-{角色}{序号}
```

示例：


| 主机名                | 含义              |
| ------------------ | --------------- |
| `prod-mom-app01`   | 生产 MOM 应用 01    |
| `prod-mom-db01`    | 生产 MOM 数据库 01   |
| `prod-mom-redis01` | 生产 MOM Redis 01 |
| `prod-mom-ntp01`   | 生产 MOM 时间服务器 01 |


设置主机名：

```bash
hostnamectl set-hostname prod-mom-app01
hostname
```

### 4.2 资产标识文件

建议在每台节点写入只读资产标识，便于脚本采集：

```bash
cat > /etc/node-info << 'EOF'
ENV=prod
BUSINESS=mom
ROLE=app
NODE_NAME=prod-mom-app01
MANAGE_IP=10.10.10.21
BUSINESS_IP=10.10.20.21
DATA_IP=
IDC=dc-a
RACK=rack-a
OWNER=ops
EOF

chmod 0644 /etc/node-info
cat /etc/node-info
```

> **注意：** `/etc/node-info` 不应存放密码、Token、私钥等敏感信息。

---

## 5. 内网 DNS 与 hosts 策略

**目的：** 解决集群节点之间名称解析一致性问题。小规模可使用 hosts，大规模必须使用内网 DNS。

### 5.1 三种方案对比


| 方案                  | 优点           | 缺点             | 适用场景        |
| ------------------- | ------------ | -------------- | ----------- |
| 方案 A：仅 `/etc/hosts` | 简单、无额外服务     | 节点多时难维护，容易漂移   | 3-10 台小集群   |
| 方案 B：内网 DNS         | 集中管理、可扩展、变更快 | 需要维护 DNS 服务高可用 | 生产推荐        |
| 方案 C：服务发现系统         | 支持健康检查和动态注册  | 引入额外组件复杂度      | 微服务、大规模动态节点 |


**推荐方案：** 生产集群使用“内网 DNS + 关键节点 hosts 兜底”。DNS 负责常规解析，`/etc/hosts` 仅保留本机、网关、DNS、NTP、堡垒机等基础设施节点。

### 5.2 hosts 兜底配置

```bash
cp /etc/hosts /etc/hosts.bak.$(date +%F_%H%M%S)

cat > /etc/hosts << 'EOF'
127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain

# ---- 基础设施节点 ----
10.10.10.10 prod-mom-bastion01
10.10.10.11 prod-mom-ntp01
10.10.10.12 prod-mom-ntp02
10.10.10.13 prod-mom-dns01
10.10.10.14 prod-mom-dns02

# ---- 当前节点 ----
10.10.20.21 prod-mom-app01
EOF
```

### 5.3 resolv.conf 配置

```bash
cp /etc/resolv.conf /etc/resolv.conf.bak.$(date +%F_%H%M%S)

cat > /etc/resolv.conf << 'EOF'
search prod.mom.local
nameserver 10.10.10.13
nameserver 10.10.10.14
options timeout:2 attempts:2 rotate
EOF

# 防止 NetworkManager 覆盖 resolv.conf；如仍使用 NetworkManager，应在网卡配置中写 DNS
chattr +i /etc/resolv.conf
```

> **注意：** 如果初始化手册中已关闭 NetworkManager，通常不会自动覆盖 `/etc/resolv.conf`。如果云厂商 DHCP 会覆盖 DNS，应改为在网卡配置或云平台侧固定 DNS。

验证：

```bash
getent hosts prod-mom-ntp01
getent hosts prod-mom-db01.prod.mom.local
nslookup prod-mom-app01.prod.mom.local 10.10.10.13
```

---

## 6. 集群时间同步体系

**目的：** 建立“外部可信时间源 → 内部时间服务器 → 集群节点”的统一时间体系。

### 6.1 推荐架构

生产内网集群不建议所有节点直接访问公网 NTP。推荐两级结构：

```text
公网/专线可信 NTP
        |
        v
prod-mom-ntp01 / prod-mom-ntp02
        |
        v
所有业务节点、数据库节点、缓存节点、日志节点、监控节点
```


| 角色            | 数量   | 要求                     |
| ------------- | ---- | ---------------------- |
| 上游时间源         | >= 3 | 阿里云、腾讯云、国家授时中心、企业专线时间源 |
| 内部 NTP Server | >= 2 | 分布在不同宿主机或故障域           |
| 普通节点          | 全部   | 只同步内部 NTP Server       |


### 6.2 时间服务器配置

在 `prod-mom-ntp01` 和 `prod-mom-ntp02` 执行：

```bash
yum install -y chrony
cp /etc/chrony.conf /etc/chrony.conf.bak.$(date +%F_%H%M%S)

cat > /etc/chrony.conf << 'EOF'
# ================================================================
# 集群内部 NTP Server 配置 - Author: 王有政
# ================================================================

# 上游时间源，建议至少 3 个
server ntp.aliyun.com iburst
server ntp1.aliyun.com iburst
server ntp.tencent.com iburst
server cn.ntp.org.cn iburst

# 允许集群内网客户端同步
allow 10.10.0.0/16

# 当本机短时间无法访问上游时，允许作为本地时间源继续服务
# stratum 10 表示优先级较低，避免覆盖真实上游时间源
local stratum 10

# 启动初期如果偏差超过 1 秒，允许前 3 次直接步进校时
makestep 1.0 3

# 同步硬件时钟
rtcsync

# NTP 服务监听地址，按实际管理网 IP 修改
bindaddress 10.10.10.11

# 日志
logdir /var/log/chrony
log measurements statistics tracking
EOF

systemctl enable chronyd --now
systemctl restart chronyd
```

`prod-mom-ntp02` 需要把 `bindaddress` 改为自身 IP：

```bash
sed -i 's/^bindaddress .*/bindaddress 10.10.10.12/' /etc/chrony.conf
systemctl restart chronyd
```

开放 NTP 服务端口：

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.0.0/16" port protocol="udp" port="123" accept'
firewall-cmd --reload
firewall-cmd --list-all
```

验证时间服务器：

```bash
chronyc sources -v
chronyc tracking
ss -lunp | grep ':123'
```

### 6.3 普通集群节点配置

所有非 NTP Server 节点执行：

```bash
yum install -y chrony
cp /etc/chrony.conf /etc/chrony.conf.bak.$(date +%F_%H%M%S)

cat > /etc/chrony.conf << 'EOF'
# ================================================================
# 集群普通节点 NTP Client 配置 - Author: 王有政
# ================================================================

# 只同步内部时间服务器
server prod-mom-ntp01 iburst prefer
server prod-mom-ntp02 iburst

# 启动初期允许快速校正
makestep 1.0 3

# 同步硬件时钟
rtcsync

# 普通节点不向其他客户端提供 NTP 服务
port 0

# 日志
logdir /var/log/chrony
EOF

systemctl enable chronyd --now
systemctl restart chronyd
```

验证：

```bash
chronyc sources -v
chronyc tracking
timedatectl status
```

### 6.4 时间偏差验收

在任一运维节点批量检查：

```bash
for host in prod-mom-app01 prod-mom-app02 prod-mom-db01 prod-mom-redis01; do
  echo "===== $host ====="
  ssh $host 'date "+%F %T.%N"; chronyc tracking | egrep "Reference ID|System time|Last offset|Leap status"'
done
```

验收标准：


| 项目            | 标准                                |
| ------------- | --------------------------------- |
| `Leap status` | `Normal`                          |
| `System time` | 绝对值建议 < 100ms，数据库和分布式锁场景建议 < 20ms |
| 上游源数量         | 至少 2 个可用                          |
| 普通节点配置        | 不直接同步公网 NTP                       |


> **重要：** 数据库、Redis Cluster、Nacos、ZooKeeper、Kafka、分布式任务调度、JWT/OAuth、证书校验都依赖稳定时间。时间异常时应先修复时间，再排查业务故障。

---

## 7. Yum 内网仓库与软件版本基线

**目的：** 保证所有节点安装的软件版本一致，避免节点之间因包版本差异产生不可预期问题。

### 7.1 三种方案对比


| 方案               | 优点            | 缺点                | 适用场景     |
| ---------------- | ------------- | ----------------- | -------- |
| 方案 A：所有节点使用公网镜像  | 简单            | 依赖公网，版本可能变化，不利于审计 | 测试环境     |
| 方案 B：云厂商内网镜像     | 快速、稳定、免公网流量   | 受云平台限制，无法完全冻结版本   | 云上中小规模集群 |
| 方案 C：自建内网 Yum 仓库 | 可冻结版本、可审计、可离线 | 需要维护仓库同步和存储       | 生产推荐     |


**推荐方案：** 生产内网集群使用自建内网 Yum 仓库；云上环境可先使用云厂商内网镜像，关键组件 RPM 应进入自建仓库。

### 7.2 客户端 repo 示例

```bash
mkdir -p /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/ 2>/dev/null || true

cat > /etc/yum.repos.d/internal.repo << 'EOF'
[centos7-base]
name=Internal CentOS 7 Base
baseurl=http://repo.prod.mom.local/centos/7/os/x86_64/
enabled=1
gpgcheck=1
gpgkey=http://repo.prod.mom.local/keys/RPM-GPG-KEY-CentOS-7

[centos7-updates]
name=Internal CentOS 7 Updates
baseurl=http://repo.prod.mom.local/centos/7/updates/x86_64/
enabled=1
gpgcheck=1
gpgkey=http://repo.prod.mom.local/keys/RPM-GPG-KEY-CentOS-7

[epel7]
name=Internal EPEL 7
baseurl=http://repo.prod.mom.local/epel/7/x86_64/
enabled=1
gpgcheck=0

[docker-ce-stable]
name=Internal Docker CE Stable
baseurl=http://repo.prod.mom.local/docker-ce/linux/centos/7/x86_64/stable/
enabled=1
gpgcheck=1
gpgkey=http://repo.prod.mom.local/keys/docker-ce.gpg
EOF

yum clean all
yum makecache
yum repolist
```

### 7.3 版本基线导出

每台节点接入后导出软件版本，便于比对：

```bash
mkdir -p /var/log/node-baseline
rpm -qa | sort > /var/log/node-baseline/rpm-qa-$(date +%F).txt
uname -a > /var/log/node-baseline/uname-$(date +%F).txt
sysctl -a 2>/dev/null | sort > /var/log/node-baseline/sysctl-$(date +%F).txt
```

---

## 8. 系统参数基线一致性

**目的：** 将单机初始化中的关键内核参数升级为集群统一基线。

### 8.1 集群通用 sysctl

```bash
cat > /etc/sysctl.d/99-cluster-baseline.conf << 'EOF'
# ================================================================
# CentOS 7.9 集群通用内核参数 - Author: 王有政
# ================================================================

# 文件句柄
fs.file-max = 2097152
fs.nr_open = 2097152

# 网络队列与连接
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 250000
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 10000 65000

# TCP keepalive，便于及时发现异常连接
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

# 禁止 ICMP 重定向和源路由
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# 内存与 swap
vm.swappiness = 10
vm.overcommit_memory = 1
vm.dirty_background_ratio = 5
vm.dirty_ratio = 20
EOF

sysctl --system
```

> **注意：** 数据库、Redis、Elasticsearch、Kafka 等组件可能有更具体的参数要求。组件手册中的角色级配置优先于通用基线，但必须记录差异。

### 8.2 limits 基线

```bash
cat > /etc/security/limits.d/99-cluster-baseline.conf << 'EOF'
# 通用进程资源限制
* soft nofile 1048576
* hard nofile 1048576
* soft nproc  65535
* hard nproc  65535

# root 保持同样限制，避免运维脚本受限
root soft nofile 1048576
root hard nofile 1048576
root soft nproc  65535
root hard nproc  65535
EOF
```

验证：

```bash
ulimit -n
ulimit -u
sysctl net.core.somaxconn fs.file-max vm.swappiness vm.overcommit_memory
```

---

## 9. 用户、sudo 与 SSH 运维通道

**目的：** 在集群规模扩大后，避免共享 root 密码、个人账号混用和 SSH 入口失控。

### 9.1 运维账号模型


| 类型   | 示例                    | 用途        | 策略           |
| ---- | --------------------- | --------- | ------------ |
| 个人账号 | `wangyouzheng`        | 日常登录和审计   | 一人一号，禁止共享    |
| 运维组  | `ops`                 | sudo 权限分组 | 通过组授权        |
| 发布账号 | `deploy`              | CI/CD 发布  | 仅允许发布目录和指定命令 |
| 服务账号 | `mysql`、`redis`、`app` | 运行服务      | 禁止交互登录       |
| 应急账号 | `breakglass`          | 堡垒机故障应急   | 强审计，默认禁用或锁定  |


### 9.2 创建运维组和发布账号

```bash
groupadd ops 2>/dev/null || true
groupadd deploy 2>/dev/null || true

useradd -m -G ops wangyouzheng 2>/dev/null || true
useradd -m -s /bin/bash deploy 2>/dev/null || true

mkdir -p /home/wangyouzheng/.ssh /home/deploy/.ssh
chmod 700 /home/wangyouzheng/.ssh /home/deploy/.ssh
chown -R wangyouzheng:wangyouzheng /home/wangyouzheng/.ssh
chown -R deploy:deploy /home/deploy/.ssh
```

### 9.3 sudo 最小授权

```bash
cat > /etc/sudoers.d/ops << 'EOF'
%ops ALL=(ALL) ALL
EOF

cat > /etc/sudoers.d/deploy << 'EOF'
deploy ALL=(root) NOPASSWD: /bin/systemctl restart app-*.service, /bin/systemctl status app-*.service, /usr/bin/journalctl
EOF

chmod 0440 /etc/sudoers.d/ops /etc/sudoers.d/deploy
visudo -cf /etc/sudoers
```

### 9.4 SSH 来源限制

集群节点应只允许堡垒机或管理网登录：

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F_%H%M%S)

cat >> /etc/ssh/sshd_config << 'EOF'

# ---- 集群运维入口限制 ----
AllowGroups ops deploy
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

sshd -t
systemctl reload sshd
```

防火墙限制 SSH 来源：

```bash
firewall-cmd --permanent --remove-service=ssh 2>/dev/null || true
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.10/32" port protocol="tcp" port="2222" accept'
firewall-cmd --reload
```

---

## 10. 网络分区与路由规范

**目的：** 保证集群内部流量路径清晰，避免业务流量、复制流量、备份流量和管理流量互相干扰。

### 10.1 网卡绑定建议


| 场景        | 建议                    |
| --------- | --------------------- |
| 单网卡小规模集群  | 通过 VLAN、安全组、防火墙实现逻辑隔离 |
| 多网卡物理机    | 管理、业务、数据、备份网络分离       |
| 高吞吐数据库/缓存 | 数据复制网络独立，避免与业务入口共享带宽  |
| 虚拟化环境     | 关注宿主机故障域，不要只看虚拟机 IP   |


### 10.2 静态路由示例

如数据网需要走独立网关：

```bash
cat > /etc/sysconfig/network-scripts/route-eth1 << 'EOF'
10.10.30.0/24 via 10.10.30.1 dev eth1
10.10.50.0/24 via 10.10.50.1 dev eth2
EOF

systemctl restart network
ip route
```

### 10.3 反向路径过滤

多网卡、多路由场景下，严格的 `rp_filter` 可能导致合法返回包被丢弃。推荐按需设置：

```bash
cat > /etc/sysctl.d/98-multinic.conf << 'EOF'
# 多网卡场景建议使用 loose 模式，避免非对称路由被误丢弃
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
EOF

sysctl --system
```

> **注意：** 如果只有单网卡且路由简单，可以保持更严格策略。多网卡调整必须配合安全组和防火墙白名单，不能以牺牲安全为代价。

---

## 11. 防火墙白名单与端口矩阵

**目的：** 将“内网互通”收敛为“按角色、按端口、按来源互通”。

### 11.1 端口矩阵模板


| 服务                | 端口             | 协议      | 来源            | 目标         | 说明                  |
| ----------------- | -------------- | ------- | ------------- | ---------- | ------------------- |
| SSH               | 2222           | TCP     | 堡垒机           | 全部节点       | 运维登录                |
| NTP               | 123            | UDP     | 全部节点          | NTP Server | 时间同步                |
| DNS               | 53             | TCP/UDP | 全部节点          | DNS Server | 内网解析                |
| Node Exporter     | 9100           | TCP     | 监控节点          | 全部节点       | 主机监控                |
| MySQL             | 3306           | TCP     | 应用节点、DB 节点    | DB 节点      | 业务访问和复制             |
| Redis             | 6379           | TCP     | 应用节点、Redis 节点 | Redis 节点   | 缓存访问                |
| Redis Cluster Bus | 16379          | TCP     | Redis 节点      | Redis 节点   | Cluster 通信          |
| Nacos             | 8848/9848/9849 | TCP     | 应用节点、Nacos 节点 | Nacos 节点   | 注册配置                |
| 日志采集              | 5044/24224     | TCP     | 全部节点          | 日志节点       | Filebeat/Fluent Bit |
| 备份                | 873/22         | TCP     | 备份节点          | 被备份节点      | rsync 或 SSH 备份      |


### 11.2 firewalld ipset 管理

```bash
# 创建基础来源集合
firewall-cmd --permanent --new-ipset=mgmt --type=hash:ip 2>/dev/null || true
firewall-cmd --permanent --new-ipset=monitor --type=hash:ip 2>/dev/null || true
firewall-cmd --permanent --new-ipset=app_nodes --type=hash:ip 2>/dev/null || true

# 添加来源 IP
firewall-cmd --permanent --ipset=mgmt --add-entry=10.10.10.10
firewall-cmd --permanent --ipset=monitor --add-entry=10.10.40.11
firewall-cmd --permanent --ipset=monitor --add-entry=10.10.40.12
firewall-cmd --permanent --ipset=app_nodes --add-entry=10.10.20.21
firewall-cmd --permanent --ipset=app_nodes --add-entry=10.10.20.22

# SSH 仅允许堡垒机
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="mgmt" port protocol="tcp" port="2222" accept'

# Node Exporter 仅允许监控节点
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="monitor" port protocol="tcp" port="9100" accept'

firewall-cmd --reload
firewall-cmd --list-all
```

### 11.3 端口连通性验证

```bash
# 从监控节点验证所有节点 Node Exporter
for host in prod-mom-app01 prod-mom-app02 prod-mom-db01; do
  nc -vz $host 9100
done

# 从普通节点验证 NTP 和 DNS
nc -vzu prod-mom-ntp01 123
nc -vzu prod-mom-dns01 53
nc -vz prod-mom-dns01 53
```

---

## 12. 节点间安全互信策略

**目的：** 明确哪些节点可以免密访问，哪些必须通过服务账号、证书或 Token 访问。

### 12.1 互信原则

1. **禁止全节点 root 互信。** root 私钥扩散会放大集群风险。
2. **发布账号只在必要节点之间互信。** 例如 CI/CD 到应用节点，不应到数据库节点。
3. **数据库、Redis 等服务复制优先使用组件级认证。** 不用系统 SSH 互信替代服务认证。
4. **备份账号只允许读取指定目录。** 不应拥有系统管理权限。
5. **所有密钥必须有负责人、用途、创建时间和轮换周期。**

### 12.2 deploy 账号免密示例

在发布机生成密钥：

```bash
sudo -u deploy ssh-keygen -t ed25519 -f /home/deploy/.ssh/id_ed25519 -C "deploy@prod-mom" -N ""
```

将公钥安装到应用节点：

```bash
mkdir -p /home/deploy/.ssh
cat >> /home/deploy/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAA... deploy@prod-mom
EOF

chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

限制 authorized_keys 来源和命令示例：

```text
from="10.10.10.20",no-agent-forwarding,no-X11-forwarding,no-port-forwarding ssh-ed25519 AAAA... deploy@prod-mom
```

> **注意：** 如需更严格控制，可使用堡垒机、集中式 IAM、LDAP/FreeIPA 或短期证书 SSH CA，而不是长期静态公钥。

---

## 13. 证书与内部 CA 管理

**目的：** 为内网 HTTPS、mTLS、服务间认证、日志采集和监控采集提供统一证书体系。

### 13.1 三种方案对比


| 方案             | 优点           | 缺点             | 适用场景       |
| -------------- | ------------ | -------------- | ---------- |
| 方案 A：自签单证书     | 快速           | 难轮换、难审计、不适合规模化 | 临时测试       |
| 方案 B：企业内部 CA   | 可控、可批量签发、可吊销 | 需要维护 CA 安全     | 生产推荐       |
| 方案 C：商业证书/ACME | 标准化程度高       | 内网域名和离线环境受限    | 互联网入口或混合环境 |


**推荐方案：** 内网服务使用企业内部 CA；公网入口使用商业证书或 ACME 自动签发。

### 13.2 分发内部 CA 根证书

```bash
mkdir -p /etc/pki/ca-trust/source/anchors
cp internal-root-ca.crt /etc/pki/ca-trust/source/anchors/internal-root-ca.crt
update-ca-trust extract

# 验证
trust list | grep -i internal -A 2
```

### 13.3 证书管理要求


| 项目     | 要求                       |
| ------ | ------------------------ |
| 私钥权限   | `0600`，仅服务运行用户可读         |
| 证书有效期  | 内网服务证书建议 90-397 天        |
| 证书 SAN | 必须包含 DNS 名称和必要 IP        |
| 轮换     | 到期前 30 天告警，到期前 7 天必须完成轮换 |
| 吊销     | 私钥泄露或人员离职时及时吊销           |


---

## 14. 内网访问边界与跳板机

**目的：** 将管理入口集中到可审计、可控制的位置。

### 14.1 访问路径

推荐访问路径：

```text
运维人员 -> VPN/办公网 -> 堡垒机/跳板机 -> 目标服务器
CI/CD    -> 发布机      -> 应用节点
监控系统 -> 监控网      -> Exporter/Agent
```

不推荐：

- 运维人员电脑直连所有服务器；
- 所有服务器开放 SSH 给整个办公网；
- 通过共享 root 密码登录；
- 数据库、Redis、Nacos 管理端口暴露给非必要网段。

### 14.2 TCP Wrapper 兜底限制

CentOS 7 上部分服务仍支持 TCP Wrapper，可作为补充，不替代防火墙：

```bash
cat > /etc/hosts.allow << 'EOF'
sshd: 10.10.10.10
EOF

cat > /etc/hosts.deny << 'EOF'
sshd: ALL
EOF
```

> **注意：** 不是所有服务都支持 TCP Wrapper。生产控制应以防火墙、安全组、堡垒机和服务自身认证为主。

---

## 15. 数据盘与目录规范

**目的：** 统一各类服务数据、日志、备份、临时文件目录，避免系统盘被写满导致整机故障。

### 15.1 目录规范


| 目录             | 用途       | 说明        |
| -------------- | -------- | --------- |
| `/data`        | 业务数据主目录  | 独立数据盘挂载   |
| `/data/mysql`  | MySQL 数据 | 数据库节点     |
| `/data/redis`  | Redis 数据 | Redis 节点  |
| `/data/app`    | 应用数据     | 应用节点      |
| `/data/logs`   | 应用日志     | 可按服务拆分    |
| `/data/backup` | 本地临时备份   | 不替代远端备份   |
| `/opt`         | 软件安装目录   | 编译安装或二进制包 |
| `/srv`         | 服务静态资源   | 可选        |


### 15.2 挂载参数建议

```bash
# 查看磁盘 UUID
blkid

# 示例：挂载 /data
mkdir -p /data
cat >> /etc/fstab << 'EOF'
UUID=<DATA_DISK_UUID> /data xfs defaults,noatime,nodiratime 0 0
EOF

mount -a
df -hT /data
```

### 15.3 权限示例

```bash
mkdir -p /data/{app,logs,backup}
chmod 0755 /data
chmod 0750 /data/app /data/logs /data/backup

# 按服务授权
chown -R app:app /data/app /data/logs
```

> **注意：** 数据库、Redis、消息队列等组件目录权限应按组件手册执行，不应统一粗暴设置为 `777`。

---

## 16. 集中日志采集

**目的：** 所有节点日志进入集中日志平台，支持故障排查、安全审计和容量分析。

### 16.1 日志类型


| 日志类型    | 路径示例                       | 采集建议  |
| ------- | -------------------------- | ----- |
| 系统日志    | `/var/log/messages`        | 必采    |
| 安全日志    | `/var/log/secure`          | 必采    |
| 审计日志    | `/var/log/audit/audit.log` | 必采    |
| cron 日志 | `/var/log/cron`            | 建议采集  |
| 应用日志    | `/data/logs/*/*.log`       | 按业务采集 |
| 组件日志    | MySQL、Redis、Nginx、Nacos    | 按角色采集 |


### 16.2 Filebeat 示例

```bash
yum install -y filebeat
cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak.$(date +%F_%H%M%S)

cat > /etc/filebeat/filebeat.yml << 'EOF'
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/messages
      - /var/log/secure
      - /var/log/cron
    fields:
      log_type: system
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /data/logs/**/*.log
    fields:
      log_type: app
    fields_under_root: true

processors:
  - add_host_metadata: ~
  - add_fields:
      target: cluster
      fields:
        env: prod
        business: mom

output.logstash:
  hosts: ["prod-mom-log01:5044", "prod-mom-log02:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
EOF

systemctl enable filebeat --now
systemctl status filebeat --no-pager
```

### 16.3 日志采集验收

```bash
logger "cluster log pipeline test from $(hostname) at $(date '+%F %T')"
sleep 10
systemctl status filebeat --no-pager
```

验收标准：


| 项目   | 标准                |
| ---- | ----------------- |
| 系统日志 | 能在日志平台按主机名检索      |
| 安全日志 | SSH 登录、sudo 记录可检索 |
| 应用日志 | 能按环境、业务、服务、节点过滤   |
| 延迟   | 普通日志延迟 < 60 秒     |
| 丢失   | Filebeat 无持续错误和积压 |


---

## 17. 监控 Agent 与告警基线

**目的：** 所有节点必须在承载生产流量前接入监控和告警。

### 17.1 Node Exporter 安装示例

```bash
useradd -r -s /sbin/nologin node_exporter 2>/dev/null || true
mkdir -p /opt/node_exporter

# 这里假设二进制已由内网制品库分发到 /tmp/node_exporter
install -m 0755 /tmp/node_exporter /opt/node_exporter/node_exporter
chown -R node_exporter:node_exporter /opt/node_exporter

cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/opt/node_exporter/node_exporter \
  --web.listen-address=:9100 \
  --collector.systemd \
  --collector.processes
Restart=always
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable node_exporter --now
systemctl status node_exporter --no-pager
```

开放监控端口：

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.40.0/24" port protocol="tcp" port="9100" accept'
firewall-cmd --reload
```

### 17.2 主机告警基线


| 指标          | 告警阈值建议            | 说明                 |
| ----------- | ----------------- | ------------------ |
| 节点存活        | `up == 0` 持续 1 分钟 | 必须告警               |
| CPU 使用率     | > 85% 持续 5 分钟     | 按角色调整              |
| 内存可用率       | < 10% 持续 5 分钟     | 数据库节点需更严格          |
| 磁盘使用率       | > 80% 预警，> 90% 严重 | `/` 和 `/data` 分别监控 |
| inode 使用率   | > 80%             | 小文件场景常见            |
| 磁盘 IO await | > 50ms 持续 5 分钟    | 数据库/日志节点重点关注       |
| 网络丢包        | > 0.1%            | 集群通信敏感             |
| 时间偏差        | > 100ms           | 分布式系统重点关注          |
| systemd 服务  | 核心服务非 active      | 按角色配置              |


### 17.3 blackbox 连通性监控

建议从监控节点对关键端口做黑盒探测：


| 目标         | 端口         | 探测                 |
| ---------- | ---------- | ------------------ |
| NTP Server | UDP 123    | NTP 同步可用           |
| DNS Server | TCP/UDP 53 | 域名解析可用             |
| 应用健康检查     | HTTP 端口    | `/health`          |
| MySQL      | 3306       | TCP connect 或 SQL  |
| Redis      | 6379       | TCP connect 或 PING |
| Nacos      | 8848       | HTTP API           |


---

## 18. 审计日志集中化

**目的：** 追踪关键文件变更、权限提升、登录行为和系统管理操作。

### 18.1 auditd 基础规则

```bash
yum install -y audit audit-libs
systemctl enable auditd --now

cat > /etc/audit/rules.d/99-cluster.rules << 'EOF'
# 身份与权限文件
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# SSH 配置和密钥
-w /etc/ssh/sshd_config -p wa -k sshd_config
-w /root/.ssh/ -p wa -k root_ssh

# 网络与名称解析
-w /etc/hosts -p wa -k hosts
-w /etc/resolv.conf -p wa -k resolv
-w /etc/sysconfig/network-scripts/ -p wa -k network

# 系统服务
-w /etc/systemd/system/ -p wa -k systemd_unit
-w /usr/lib/systemd/system/ -p wa -k systemd_unit

# 时间配置
-w /etc/chrony.conf -p wa -k chrony
EOF

augenrules --load
auditctl -s
```

### 18.2 sudo 日志独立文件

```bash
cat > /etc/sudoers.d/00-logging << 'EOF'
Defaults logfile="/var/log/sudo.log"
Defaults log_input,log_output
EOF

chmod 0440 /etc/sudoers.d/00-logging
visudo -cf /etc/sudoers
```

### 18.3 审计查询

```bash
ausearch -k sshd_config
ausearch -k sudoers
ausearch -m USER_LOGIN -ts today
```

---

## 19. 备份、快照与恢复演练

**目的：** 集群节点接入生产前必须明确哪些数据需要备份、备份到哪里、如何恢复。

### 19.1 备份分层


| 层级   | 示例                                  | 策略             |
| ---- | ----------------------------------- | -------------- |
| 系统配置 | `/etc`、systemd unit、sysctl、firewall | 每日备份，变更前备份     |
| 应用配置 | Nginx、应用配置、启动参数                     | 每次发布归档         |
| 业务数据 | MySQL、Redis RDB/AOF、文件数据            | 按组件制定 RPO/RTO  |
| 日志数据 | 安全日志、审计日志、业务日志                      | 集中采集，按合规保留     |
| 制品   | RPM、JAR、镜像、脚本                       | 制品库保留，不依赖服务器本地 |


### 19.2 系统配置备份示例

```bash
mkdir -p /data/backup/system

tar --xattrs --acls -czf /data/backup/system/etc-$(hostname)-$(date +%F_%H%M%S).tar.gz \
  /etc \
  /usr/lib/systemd/system \
  /etc/systemd/system

sha256sum /data/backup/system/etc-*.tar.gz > /data/backup/system/SHA256SUMS
```

同步到备份服务器：

```bash
rsync -avz --delete /data/backup/system/ backup@prod-mom-backup01:/data/backup/system/$(hostname)/
```

### 19.3 恢复演练要求


| 项目   | 要求                    |
| ---- | --------------------- |
| 演练频率 | 生产核心系统至少每季度一次         |
| 演练环境 | 优先在隔离测试环境恢复           |
| 验证内容 | 配置完整性、服务启动、数据一致性、权限正确 |
| 记录   | 必须记录耗时、问题、改进项         |


> **注意：** 没有恢复演练的备份不应视为有效备份。

---

## 20. 配置管理与批量变更

**目的：** 避免集群节点越多，配置漂移越严重。

### 20.1 三种方案对比


| 方案                     | 优点         | 缺点        | 适用场景  |
| ---------------------- | ---------- | --------- | ----- |
| 方案 A：手工 SSH 执行         | 灵活         | 不可追溯、易遗漏  | 临时排障  |
| 方案 B：批量脚本              | 简单，可记录     | 幂等性较弱     | 小规模集群 |
| 方案 C：Ansible/SaltStack | 幂等、可审计、可复用 | 需要规范目录和流程 | 生产推荐  |


**推荐方案：** 使用 Ansible 管理系统基线、用户、SSH、防火墙、chrony、监控 Agent、日志 Agent。

### 20.2 inventory 示例

```ini
[ntp]
prod-mom-ntp01 ansible_host=10.10.10.11
prod-mom-ntp02 ansible_host=10.10.10.12

[app]
prod-mom-app01 ansible_host=10.10.20.21
prod-mom-app02 ansible_host=10.10.20.22

[db]
prod-mom-db01 ansible_host=10.10.30.21
prod-mom-db02 ansible_host=10.10.30.22

[redis]
prod-mom-redis01 ansible_host=10.10.30.31
prod-mom-redis02 ansible_host=10.10.30.32

[all:vars]
ansible_user=wangyouzheng
ansible_port=2222
ansible_become=true
```

### 20.3 批量验证命令

```bash
ansible all -m ping
ansible all -a 'hostname; date; chronyc tracking | grep "Leap status"'
ansible all -a 'df -h / /data'
ansible all -a 'systemctl is-active node_exporter filebeat chronyd'
```

---

## 21. 服务发现与负载均衡接入

**目的：** 让节点加入集群后能够被正确发现、调度和摘除。

### 21.1 接入方式


| 类型     | 示例                     | 适用场景          |
| ------ | ---------------------- | ------------- |
| DNS 解析 | `app.prod.mom.local`   | 稳定服务入口        |
| 负载均衡   | Nginx、HAProxy、SLB      | HTTP/TCP 流量入口 |
| 注册中心   | Nacos、Consul、ZooKeeper | 微服务动态发现       |
| 配置中心   | Nacos、Apollo           | 配置集中管理        |


### 21.2 节点上线流程

1. 节点完成初始化和本文集群增量配置；
2. 接入监控、日志和审计；
3. 完成端口白名单配置；
4. 部署业务服务但不接流量；
5. 本机健康检查通过；
6. 加入注册中心或负载均衡后端；
7. 小流量验证；
8. 完整接入生产流量。

### 21.3 节点下线流程

1. 从负载均衡或注册中心摘除；
2. 等待连接自然排空；
3. 停止业务服务；
4. 保留监控和日志采集一段时间；
5. 如为数据节点，先完成副本迁移、主从切换或数据备份；
6. 更新资产登记和 DNS；
7. 回收账号、证书、密钥和防火墙规则。

---

## 22. 变更窗口与回滚策略

**目的：** 集群配置变更必须可控、可验证、可回滚。

### 22.1 变更前检查

```bash
mkdir -p /data/backup/change-$(date +%F_%H%M%S)
CHANGE_DIR=$(ls -td /data/backup/change-* | head -1)

cp -a /etc/chrony.conf $CHANGE_DIR/ 2>/dev/null || true
cp -a /etc/hosts $CHANGE_DIR/ 2>/dev/null || true
cp -a /etc/resolv.conf $CHANGE_DIR/ 2>/dev/null || true
cp -a /etc/sysctl.d $CHANGE_DIR/sysctl.d 2>/dev/null || true
cp -a /etc/firewalld $CHANGE_DIR/firewalld 2>/dev/null || true
cp -a /etc/ssh/sshd_config $CHANGE_DIR/ 2>/dev/null || true

systemctl list-units --type=service --state=running > $CHANGE_DIR/running-services.txt
ss -lntup > $CHANGE_DIR/listen-ports.txt
ip addr > $CHANGE_DIR/ip-addr.txt
ip route > $CHANGE_DIR/ip-route.txt
```

### 22.2 变更策略


| 变更类型       | 策略                    |
| ---------- | --------------------- |
| 时间同步       | 先改 NTP Server，再灰度普通节点 |
| DNS        | 先降低 TTL，再切换，再观察       |
| 防火墙        | 先添加新规则，验证后删除旧规则       |
| SSH        | 保留一个已登录会话，验证新连接后再退出   |
| sysctl     | 先运行时生效验证，再固化文件        |
| 监控日志 Agent | 可批量，但必须观察服务端压力        |


### 22.3 回滚原则

1. 每次变更必须有明确回滚命令；
2. 防火墙、SSH、网络变更必须优先保证不失联；
3. 涉及数据节点时，回滚前必须判断是否会造成数据回退或脑裂；
4. 回滚后必须重新执行验收清单。

---

## 23. 故障域与高可用部署规范

**目的：** 避免“看起来有多副本，实际都在同一故障点”。

### 23.1 故障域维度


| 故障域  | 示例           | 要求         |
| ---- | ------------ | ---------- |
| 物理机  | 同一宿主机上的多台虚拟机 | 主从不共宿主机    |
| 机柜   | 同一机柜交换机或电源   | 核心副本跨机柜    |
| 可用区  | 云上 AZ        | 关键服务跨 AZ   |
| 网络设备 | 同一交换机/路由器    | 核心链路冗余     |
| 存储   | 同一存储阵列       | 数据副本避免共用单点 |
| 运维入口 | 单堡垒机         | 至少有应急入口    |


### 23.2 常见组件部署建议


| 组件            | 最小生产建议             | 故障域要求                  |
| ------------- | ------------------ | ---------------------- |
| NTP           | 2 节点               | 分散宿主机                  |
| DNS           | 2 节点               | 分散宿主机                  |
| Nginx/网关      | 2 节点               | 分散宿主机，可配 VIP/LB        |
| 应用服务          | >= 2 节点            | 分散宿主机                  |
| MySQL 主从      | 1 主 1 从 + 备份       | 主从分散，备份独立              |
| Redis 主从      | 1 主 1 从 + Sentinel | Sentinel 至少 3 节点       |
| Redis Cluster | 3 主 3 从            | replica 不与 master 同故障域 |
| Nacos         | 3 节点               | 奇数节点，分散部署              |
| Prometheus    | 1-2 节点             | 关键场景双写或远端存储            |
| 日志集群          | >= 3 节点            | 数据副本跨故障域               |


---

## 24. 容量规划与压测准入

**目的：** 节点加入集群前确认资源足够，避免上线后立即成为瓶颈。

### 24.1 容量规划项


| 资源   | 规划内容                    |
| ---- | ----------------------- |
| CPU  | 峰值 QPS、压缩、加密、序列化开销      |
| 内存   | 应用堆、Page Cache、连接数、缓冲区  |
| 磁盘   | 数据增长、日志增长、备份空间、保留周期     |
| IOPS | 数据库、日志、消息队列写入压力         |
| 网络   | 入口流量、复制流量、备份流量、监控日志流量   |
| 连接数  | 客户端连接、长连接、短连接、TIME_WAIT |
| 文件句柄 | 日志文件、socket、数据文件        |


### 24.2 上线准入建议


| 项目     | 建议标准                       |
| ------ | -------------------------- |
| CPU 峰值 | 压测峰值下 < 70%                |
| 内存     | 压测峰值下可用内存 > 20%            |
| 磁盘     | 上线时使用率 < 60%，预测 6 个月 < 80% |
| 网络     | 峰值带宽 < 网卡能力 50%            |
| 延迟     | 满足业务 SLA，P99 不超过设计阈值       |
| 错误率    | 压测期间无系统性错误                 |
| 故障演练   | 至少完成一次节点摘除或重启演练            |


### 24.3 基础压测工具

```bash
# 网络吞吐，服务端
iperf3 -s

# 网络吞吐，客户端
iperf3 -c prod-mom-app02 -t 30 -P 4

# 磁盘顺序写入参考，谨慎在生产低峰执行
fio --name=seqwrite --filename=/data/fio-test.dat --size=2G --bs=1M --rw=write --direct=1 --iodepth=16 --numjobs=1 --runtime=60 --time_based --group_reporting

# 磁盘随机读写参考
fio --name=randrw --filename=/data/fio-test.dat --size=2G --bs=4k --rw=randrw --rwmixread=70 --direct=1 --iodepth=32 --numjobs=4 --runtime=60 --time_based --group_reporting

rm -f /data/fio-test.dat
```

> **注意：** 压测可能影响生产节点性能，应在上线前或维护窗口执行，并明确测试范围。

---

## 25. 最终验收清单

### 25.1 单节点接入验收


| 检查项      | 命令                                  | 标准                    |
| -------- | ----------------------------------- | --------------------- |
| 主机名      | `hostname`                          | 符合命名规范                |
| 资产标识     | `cat /etc/node-info`                | 信息完整且无敏感数据            |
| DNS      | `getent hosts prod-mom-ntp01`       | 可解析                   |
| 时间       | `chronyc tracking`                  | `Leap status: Normal` |
| Yum      | `yum repolist`                      | 指向内网源或指定源             |
| sysctl   | `sysctl --system`                   | 无错误                   |
| limits   | `ulimit -n`                         | 达到基线                  |
| SSH      | `sshd -t`                           | 配置合法                  |
| 防火墙      | `firewall-cmd --list-all`           | 仅开放必要端口               |
| 数据盘      | `df -hT /data`                      | 独立挂载                  |
| 日志 Agent | `systemctl is-active filebeat`      | `active`              |
| 监控 Agent | `systemctl is-active node_exporter` | `active`              |
| 审计       | `auditctl -s`                       | enabled               |
| 备份       | `rsync` 或备份任务                       | 能成功备份到远端              |


### 25.2 集群级验收


| 检查项  | 标准                           |
| ---- | ---------------------------- |
| 时间同步 | 全部节点偏差 < 100ms，核心数据节点 < 20ms |
| 名称解析 | 所有节点能解析基础设施和同角色节点            |
| 端口矩阵 | 来源、目标、端口与规划一致                |
| 监控   | 节点在监控平台可见，核心告警已配置            |
| 日志   | 系统、安全、应用日志可集中检索              |
| 审计   | SSH、sudo、关键配置变更可追踪           |
| 备份   | 至少完成一次备份和恢复验证                |
| 故障域  | 主从、副本、仲裁不在同一故障域              |
| 变更记录 | 所有集群级配置有记录和回滚方案              |
| 安全边界 | 普通办公网不能直接访问核心数据端口            |


### 25.3 批量验收脚本

```bash
for host in prod-mom-app01 prod-mom-app02 prod-mom-db01 prod-mom-redis01; do
  echo "==================== $host ===================="
  ssh $host '
    echo "[hostname]"; hostname
    echo "[time]"; chronyc tracking | egrep "System time|Leap status"
    echo "[disk]"; df -hT / /data 2>/dev/null
    echo "[service]"; systemctl is-active chronyd node_exporter filebeat auditd 2>/dev/null
    echo "[firewall]"; firewall-cmd --state 2>/dev/null
    echo "[listening]"; ss -lntup | egrep "(:2222|:9100|:6379|:3306|:8848)" || true
  '
done
```

---

## 附录 A：单台服务器融入集群推荐流程

1. 完成 `CentOS7.9生产环境初始化手册.md`；
2. 登记资产信息、IP、角色、故障域；
3. 设置规范主机名和 `/etc/node-info`；
4. 配置内网 DNS 或 hosts 兜底；
5. 接入内部 NTP 时间体系；
6. 切换到内网 Yum 仓库或指定版本基线；
7. 应用集群 sysctl 和 limits 基线；
8. 配置运维账号、sudo、SSH 来源限制；
9. 按端口矩阵配置防火墙；
10. 挂载数据盘并创建服务目录；
11. 安装并接入日志 Agent；
12. 安装并接入监控 Agent；
13. 配置 auditd 审计规则；
14. 配置系统配置备份；
15. 执行单节点验收；
16. 加入服务发现、注册中心或负载均衡；
17. 小流量验证；
18. 正式接入生产流量。

---

## 附录 B：常见问题

### B.1 为什么不能所有节点直接同步公网 NTP？

原因如下：

- 内网服务器可能无公网访问能力；
- 公网 NTP 质量受网络波动影响；
- 每台节点出口策略不同，时间源可能不一致；
- 无法统一审计和控制；
- 当公网异常时，整个集群可能出现不同程度漂移。

生产推荐使用内部 NTP Server 作为统一时间源。

### B.2 是否必须自建 DNS？

不是必须，但生产推荐。小规模集群可以先使用 `/etc/hosts`，但超过 10 台节点或存在频繁扩缩容时，应建设内网 DNS，否则 hosts 漂移会成为严重隐患。

### B.3 初始化手册已经配置了公网 NTP，是否冲突？

不算错误。初始化阶段使用公网 NTP 可以保证单机时间正确；融入集群后，应改为同步内部 NTP Server。本文作为增量文档，集群时间体系优先。

### B.4 内网集群是否可以关闭防火墙？

不推荐。内网不是安全边界。即使云安全组已经限制，也建议保留主机防火墙作为第二道防线。若因性能或组件兼容性必须关闭 firewalld，应有等效安全组、ACL 或主机级 iptables 规则，并记录审批原因。

### B.5 是否需要所有节点互相 SSH 免密？

不需要，也不推荐。只有发布机、备份机、配置管理节点等明确需要访问目标节点的场景，才配置受限免密。数据库复制、Redis Cluster 通信、Nacos 集群通信应使用组件自身认证和端口白名单，不应依赖 SSH 互信。