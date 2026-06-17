# CentOS 7.9 生产服务器初始化手册

> **Author:** 王有政
> **日期:** 2026-06-15
> **适用系统:** CentOS 7.9 (x86_64)
> **说明:** 本文档按优先级与配置依赖排序，每一步均可独立执行并验证，避免一键脚本中途失败无法定位问题。

---

## 目录

### 前置检查

1. [硬件与系统信息确认](#1-硬件与系统信息确认)

### 系统基础（P0）

1. [Yum 镜像源配置](#2-yum-镜像源配置)
2. [基础工具安装](#3-基础工具安装)
3. [系统补丁更新](#4-系统补丁更新)
4. [Hostname 与 hosts 配置](#5-hostname-与-hosts-配置)
5. [时区与时间同步](#6-时区与时间同步)
6. [关闭不必要服务](#7-关闭不必要服务)

### 安全加固（P0 · 生产最高优先级）

1. [用户与密码策略加固](#8-用户与密码策略加固)
2. [SSH 安全加固](#9-ssh-安全加固)
3. [SELinux 配置](#10-selinux-配置)
4. [主机防火墙基线](#11-主机防火墙基线)
5. [fail2ban 防暴力破解](#12-fail2ban-防暴力破解)
6. [/tmp 分区加固](#13-tmp-分区加固)
7. [auditd 审计日志](#14-auditd-审计日志)

### 内核与性能调优（P1 · 高优先级）

1. [Swap 与内存管理](#15-swap-与内存管理)
2. [Transparent Huge Pages (THP) 配置](#16-transparent-huge-pages-thp-配置)
3. [挂载参数优化](#17-挂载参数优化)
4. [磁盘 I/O 调度器配置](#18-磁盘-io-调度器配置)
5. [内核参数优化 (sysctl)](#19-内核参数优化-sysctl)
6. [文件句柄数限制](#20-文件句柄数限制)
7. [网络接口优化](#21-网络接口优化)
8. [SysRq 紧急调试开关](#22-sysrq-紧急调试开关)

### 运行环境（P2 · 中优先级）

1. [JDK 安装与环境变量设置](#23-jdk-安装与环境变量设置)
2. [常用诊断工具安装](#24-常用诊断工具安装)
3. [Docker 安装与配置](#25-docker-安装与配置)
4. [日志配置](#26-日志配置)
5. [Core Dump 配置](#27-core-dump-配置)

### 验收

1. [最终验证清单](#28-最终验证清单)

---

## 1. 硬件与系统信息确认

**目的：** 确认硬件配置满足生产最低要求。

```bash
# 系统版本
cat /etc/redhat-release
uname -r

# CPU 信息（建议 >= 4 核）
lscpu | grep -E "^CPU\(s\)|^Model name|^Thread"

# 内存（建议 >= 8G）
free -h

# 磁盘（建议根分区 >= 50G，数据盘独立挂载）
lsblk
fdisk -l
df -h

# 网卡信息
ip addr show

# ---- 网络带宽与质量确认 ----
# 网卡协商速率
ethtool eth0 2>/dev/null | grep -E "Speed|Duplex|Link"

# 到网关的延迟与丢包率（-c 100 发 100 个包）
gateway=$(ip route | awk '/^default/{print $3}')
echo "网关: $gateway"
ping -c 100 -i 0.2 "$gateway" | tail -2

# 到外网的延迟参考
ping -c 10 -i 0.2 223.5.5.5 | tail -2

# 下行带宽简易测试（下载阿里云镜像文件，观察速度）
# Ctrl+C 手动停止，观察平均下载速度
curl -o /dev/null -w "下载速度: %{speed_download} bytes/s\n" https://mirrors.aliyun.com/centos/7/os/x86_64/repodata/repomd.xml

# 上行带宽测试（需对端有接收服务，满足条件时执行）
# 方案A: iperf3（需对端运行 iperf3 -s）
# iperf3 -c <对端IP> -t 10 -P 4
#
# 方案B: scp 大文件到对端服务器
# dd if=/dev/zero of=/tmp/test_upload bs=1M count=100
# scp /tmp/test_upload user@<对端IP>:/tmp/
# rm -f /tmp/test_upload
```

**验收标准：**


| 项目       | 最低要求       | 推荐配置       |
| -------- | ---------- | ---------- |
| CPU      | 4 核        | 8 核+       |
| 内存       | 8G         | 16G+       |
| 系统盘      | 50G SSD    | 100G SSD   |
| 数据盘      | 独立挂载 /data | 200G+ SSD  |
| 网卡协商速率   | 千兆         | 万兆（数据库服务器） |
| 丢包率（到网关） | < 0.1%     | 0%         |
| 网关延迟     | < 1ms      | < 0.3ms    |
| 下行带宽     | >= 100MB/s | >= 500MB/s |


> **注意：** 如果内存不满足要求，需要先扩容再继续。数据盘务必独立于系统盘，方便后续扩容和故障隔离。

### 1.1 生产执行前风险确认

**目的：** 避免把“新装空白服务器初始化命令”直接套到已有业务、云主机、专线/VPN、Docker/Kubernetes 节点上，导致 SSH 锁死、网络中断、路由冲突或服务异常。

```bash
# 服务器角色与网络环境确认
hostname
cat /etc/redhat-release
uname -r
ip addr
ip route
cat /etc/resolv.conf
systemctl is-active NetworkManager firewalld docker chronyd 2>/dev/null

# 是否已有关键业务进程
ps -eo pid,ppid,user,comm,args --sort=comm | head -50
ss -tulnp

# 磁盘与挂载确认，避免误格式化或错误修改 fstab
df -hT
lsblk -f
findmnt

# 云主机/虚拟化环境参考信息
dmidecode -s system-product-name 2>/dev/null || true
```

**执行策略：**

| 场景 | 执行原则 |
| --- | --- |
| 新装空白服务器 | 可按本文档顺序执行，但仍需确认云厂商、DNS、NTP、Yum、Docker 网段 |
| 已有业务服务器 | 禁止直接整段复制执行，必须逐项评估影响并安排维护窗口 |
| 云主机/ECS/CVM | 不要随意关闭 NetworkManager、修改 SSH 端口、覆盖 DNS/NTP；主机防火墙按本文第 11 节统一基线执行 |
| VPN/专线/多 VPC/CEN 环境 | 必须先拿到企业网段规划，重点检查 Docker、K8s、内网 DNS 路由冲突 |
| Docker/K8s 节点 | `ip_forward`、`rp_filter`、firewalld/iptables、Docker 网段必须按容器主机场景配置 |
| 数据库/Redis 服务器 | Swap、THP、ulimit、I/O 调度器可更激进，但必须在维护窗口执行 |

> **强制要求：** 涉及 SSH、防火墙、NetworkManager、`/etc/fstab`、`sysctl`、Docker 网络、SELinux 的配置，必须先备份、再验证、最后保留回滚路径。生产环境不要把 `cat >`、`cat >>`、`sed -i` 当作可无限重复执行的幂等命令。

### 1.2 数据盘挂载点与目录规范

**目的：** 统一生产服务器数据盘目录语义，避免软件安装目录、业务数据目录、日志目录和备份目录混用。

本文统一以 `/data` 作为示例数据盘挂载点。`/data` 只是挂载点名称，不是强制标准；如果实际服务器数据盘挂载到 `/data1`、`/mnt/data`、`/u01` 等路径，应按现场挂载点整体替换，并在部署文档和资产台账中保持一致。

推荐目录规范：

| 目录 | 用途 | 示例 |
|---|---|---|
| `/data/module` | 软件安装目录、版本目录、当前版本软链接，以及应用自包含目录 | `/data/module/mysql`、`/data/module/redis7.2` |
| `/data/soft` | 软件包、安装包、离线依赖包暂存目录，可选 | `/data/soft/*.tar.gz` |
| `/data/module/{应用}` | 单个应用的配置、数据、日志、运行时文件、备份聚合目录 | `/data/module/redis7.2/{conf,data,logs,run,backup}` |

目录使用原则：

1. `/data` 只是示例数据盘挂载点，不代表所有服务器都必须叫 `/data`；
2. 单个应用的配置、数据、日志、运行时文件和备份优先聚合在自己的应用目录下；
3. Redis 等中间件可采用 `/data/module/{当前版本软链接}/{conf,data,logs,run,backup}`；
4. 软件包、源码包、离线依赖包可临时放在 `/data/soft`，也可按团队现状放在 `/data/module`，但应避免与运行数据混淆；
5. 生产服务进程不应拥有配置文件写权限，配置文件建议 `root:{服务用户}`、`640` 或按服务要求最小授权。

示例：

```text
/data/module/redis7.2.14/
/data/module/redis7.2 -> /data/module/redis7.2.14
/data/module/redis7.2/conf
/data/module/redis7.2/data
/data/module/redis7.2/logs
/data/module/redis7.2/run
/data/module/redis7.2/backup
```

> **注意：** 历史服务器如果已经使用 `/etc`、`/data/data`、`/data/logs`、`/run` 等旧目录，生产环境不要为了目录统一强行迁移。新部署按本文规范执行；历史实例可在维护窗口评估迁移收益、风险和回滚方案后再处理。

---

## 2. Yum 镜像源配置

**目的：** CentOS 7 已于 2024 年 6 月 EOL，官方源已归档。必须按云厂商、公网、企业内网三类场景选择可用源，否则后续所有 yum 操作均会失败。

> **实战避坑：** 不要默认认为 `mirrors.aliyun.com/centos/7/` 永远可用。CentOS 7 EOL 后，很多镜像站会迁移到归档路径。内网服务器还可能完全无法访问公网源。执行前先确认 DNS、网络出口、云厂商内网源、企业内网源策略。

```bash
# 备份原有 repo
mkdir -p /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/

# ============================================================
# 以下全部手工编写 repo 文件，直接指向阿里云镜像。
# 无需先 yum install epel-release，因为它的作用就是生成
# epel.repo 文件，和手工写效果一致，且无网络依赖。
# ============================================================

# CentOS 7 基础源（Base + Updates + Extras）
cat > /etc/yum.repos.d/CentOS-Base.repo << 'EOF'
[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.aliyun.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
enabled=1

[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
enabled=1

[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
enabled=1
EOF

# EPEL 源也指向阿里云镜像
cat > /etc/yum.repos.d/epel.repo << 'EOF'
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=https://mirrors.aliyun.com/epel/7/$basearch/
gpgcheck=0
enabled=1
EOF

# Docker CE 源
cat > /etc/yum.repos.d/docker-ce.repo << 'EOF'
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
enabled=1
EOF

# 清理并重建缓存
yum clean all
yum makecache

# 验证源是否可用
yum repolist
yum list --showduplicates | head -5
```

**可选镜像源对比：**


| 镜像源     | 地址                             | 适用场景     |
| ------- | ------------------------------ | -------- |
| 阿里云（推荐） | `mirrors.aliyun.com`           | 国内通用，速度快 |
| 腾讯云     | `mirrors.cloud.tencent.com`    | 腾讯云服务器优选 |
| 华为云     | `mirrors.huaweicloud.com`      | 华为云服务器优选 |
| 清华大学    | `mirrors.tuna.tsinghua.edu.cn` | 教育网内优选   |


> **注意：** 如果服务器在云平台上（阿里云/腾讯云等），应使用对应云平台的内部镜像源，走内网免流量费且延迟更低。

---

## 3. 基础工具安装

**目的：** 配好 yum 后立即安装基础工具。`vim`、`rz`/`sz`、`wget` 等全程依赖，越早装越好，避免后续操作受阻。

```bash
yum install -y \
    vim \
    wget \
    curl \
    lrzsz \
    unzip \
    zip \
    net-tools \
    bash-completion \
    tree \
    git \
    rsync

# 启用 bash 自动补全
source /etc/profile.d/bash_completion.sh
```

> **注意：** 诊断类工具（htop、iotop、strace 等）在后续"常用诊断工具安装"章节单独安装，此处只装操作必备的基础工具。

---

## 4. 系统补丁更新

**目的：** 修复已知安全漏洞，确保系统处于最新状态。

```bash
# 查看可用更新
yum check-update

# 更新所有包（生产环境建议在维护窗口操作）
yum update -y

# 更新安全补丁（如果不想全量更新，可仅更新安全补丁）
# yum update --security -y

# 更新完成后重启（如果涉及内核更新）
reboot
```

> **注意：** 全量更新可能影响已有服务，建议先在测试环境验证后再执行。若涉及内核更新，必须重启。

---

## 5. Hostname 与 hosts 配置

**目的：** 设置规范的主机名，便于运维管理和日志排查。

```bash
# 设置主机名（按企业命名规范修改）
# 命名规范建议: {环境}-{业务}-{编号}
# 示例: prod-mom-web01, prod-mom-db01
hostnamectl set-hostname prod-mom-app01

# 配置 hosts 文件
cat >> /etc/hosts << 'EOF'
# ---- 生产服务器节点 ----
192.168.1.10    prod-mom-app01
192.168.1.11    prod-mom-app02
192.168.1.20    prod-mom-db01
192.168.1.30    prod-mom-redis01
192.168.1.40    prod-mom-nacos01
EOF

# 验证
hostname
cat /etc/hosts
```

---

## 6. 时区与时间同步

**目的：** 确保所有服务器时间一致，避免分布式场景下 token/签名校验失败。

```bash
# 设置时区
timedatectl set-timezone Asia/Shanghai

# 开启 NTP 同步
timedatectl set-ntp yes

# 安装 chrony（CentOS 7 推荐用 chrony 替代 ntpd）
yum install -y chrony

# 配置 chrony（使用国内 NTP 源）
cat > /etc/chrony.conf << 'EOF'
# 阿里云 NTP 服务器
server ntp.aliyun.com iburst
server ntp1.aliyun.com iburst
server ntp2.aliyun.com iburst
server cn.ntp.org.cn iburst

# 时间偏差超过 1 秒时允许步进式调整
makestep 1.0 3

# 启用 RTC 同步
rtcsync

# 日志目录
logdir /var/log/chrony
EOF

# 启动 chrony
systemctl enable chronyd --now

# 验证
chronyc sources -v
timedatectl status
```

---

## 7. 关闭不必要服务

**目的：** 减少攻击面，释放系统资源。

> **实战避坑：** 云主机、DHCP、多网卡、弹性网卡、VLAN、bond、策略路由环境可能依赖 NetworkManager。不要默认执行 `systemctl stop NetworkManager`，否则可能直接造成 SSH 断连、默认路由丢失或 DNS 失效。

```bash
# 先确认当前网络是否由 NetworkManager 管理
systemctl is-active NetworkManager
nmcli dev status 2>/dev/null || true
ip route
cat /etc/resolv.conf
ls /etc/sysconfig/network-scripts/ifcfg-* 2>/dev/null

# 仅当确认网卡使用传统 network-scripts 管理，且不是云厂商依赖 NetworkManager 的镜像时，才考虑关闭
# systemctl stop NetworkManager
# systemctl disable NetworkManager
# systemctl enable network --now

# 关闭明确不需要的服务
for svc in postfix avahi-daemon cups bluetooth; do
    systemctl stop $svc 2>/dev/null
    systemctl disable $svc 2>/dev/null
done

# 查看当前运行的服务，确认无多余服务
systemctl list-units --type=service --state=running
```

> **建议：** 生产云主机默认保留 NetworkManager，除非网络配置已固化、具备控制台/VNC/云助手回滚手段，并在维护窗口验证。

---

## 8. 用户与密码策略加固

**目的：** 防止弱密码和账号暴力破解。**生产环境必须最先加固的安全项。**

```bash
# 安装密码策略模块
yum install -y pam_pwquality

# 配置密码复杂度
cat > /etc/security/pwquality.conf << 'EOF'
# 密码最小长度 12 位
minlen = 12
# 至少包含 1 个大写字母
ucredit = -1
# 至少包含 1 个小写字母
lcredit = -1
# 至少包含 1 个数字
dcredit = -1
# 至少包含 1 个特殊字符
ocredit = -1
# 密码中不能包含用户名
usercheck = 1
# 新密码与旧密码至少 8 个字符不同
difok = 8
EOF

# 配置登录失败锁定策略
cat > /etc/security/faillock.conf << 'EOF'
# 连续 5 次失败后锁定
deny = 5
# 锁定时间 15 分钟（900秒）
unlock_time = 900
# 统计失败的时间窗口 15 分钟
fail_interval = 900
EOF

# 密码有效期设置
# 修改 /etc/login.defs
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs    # 密码最长有效期 90 天
sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs     # 密码最短使用期 1 天
sed -i 's/^PASS_MIN_LEN.*/PASS_MIN_LEN    12/' /etc/login.defs     # 密码最小长度 12
sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   14/' /etc/login.defs    # 到期前 14 天提醒

# 禁止空密码登录
sed -i 's/^nullok//g' /etc/pam.d/system-auth

# 设置账户超时自动注销（30 分钟无操作）
echo "TMOUT=1800" >> /etc/profile
echo "readonly TMOUT" >> /etc/profile
echo "export TMOUT" >> /etc/profile
```

---

## 9. SSH 安全加固

**目的：** 减少 SSH 攻击面，兼顾 DataX/Hive 等 SSH 组件调用的自动化运维场景。

### 9.1 SSH 主配置

> **实战避坑：** SSH 是最容易把自己锁在服务器外的配置项。生产环境不建议直接全量覆盖 `/etc/ssh/sshd_config`，尤其是云主机、堡垒机、Ansible、JumpServer、备份系统、DataX/Hive/SFTP 调度节点依赖 SSH 的场景。修改端口、禁用密码、限制算法必须分阶段灰度执行。

```bash
# 备份原配置，备份文件带时间戳
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.$(date +%F_%H%M%S).bak

# 修改前确认当前登录方式、云安全组/硬件防火墙/堡垒机策略
ss -tulnp | grep sshd
who
last -n 5

# 建议人工编辑或使用受控配置管理工具合并，不建议直接 cat > 覆盖整个文件
# vim /etc/ssh/sshd_config

# 推荐逐步调整的关键项：
# Port 2222
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes
# MaxAuthTries 6
# MaxSessions 100
# MaxStartups 100:150:200
# ClientAliveInterval 300
# ClientAliveCountMax 6
# TCPKeepAlive yes
# UseDNS no
# GSSAPIAuthentication no
# Banner /etc/ssh/banner

# 创建登录 Banner
cat > /etc/ssh/banner << 'EOF'
*******************************************************************
*                    WARNING: Authorized Access Only              *
*     All connections are monitored and recorded. Disconnect      *
*     IMMEDIATELY if you are not an authorized user.              *
*******************************************************************
EOF

# 语法校验，必须通过后才能 reload/restart
sshd -t

# 优先 reload，避免直接 restart 导致现有连接受影响
systemctl reload sshd

# 新开终端验证，原 SSH 会话不要断开
# ssh -p 2222 ops@<服务器IP>
# sudo -l
```

**强制验证顺序：**

1. 云安全组/硬件防火墙/堡垒机已放行新 SSH 端口。
2. 至少一个非 root 运维账号可使用密钥登录。
3. 该账号具备必要 sudo 权限。
4. 当前 SSH 会话保持不断开。
5. `sshd -t` 语法校验通过。
6. 使用新终端验证新端口、新账号、sudo 权限。
7. 验证成功后，再考虑禁用密码登录或关闭旧端口。

> **警告：** `Ciphers`、`MACs`、`KexAlgorithms` 不建议默认收得过窄，老堡垒机、SFTP 工具、自动化客户端可能不兼容。需要合规加固时，应先盘点所有 SSH 客户端能力，再逐步收敛算法。

### 9.2 关键参数说明（兼顾自动化场景）


| 参数                    | 值           | 说明                                                     |
| --------------------- | ----------- | ------------------------------------------------------ |
| `MaxSessions`         | 100         | 已认证最大并发会话数（DataX 40路 + Hive 10路 + 人工5路 + 余量45）         |
| `MaxStartups`         | 100:150:200 | 握手阶段并发限制，前100个不拒绝                                      |
| `ClientAliveCountMax` | 6           | 配合 `ClientAliveInterval 300` 可容忍单连接30分钟无数据，Hive长查询不会断连 |
| `TCPKeepAlive`        | yes         | 操作系统层面保活，防止中间防火墙/NAT设备切断空闲连接                           |
| `MaxAuthTries`        | 6           | 多密钥环境下允许更多次尝试（自动化的机器可能持有多把密钥轮试）                        |
| `LoginGraceTime`      | 90          | 给自动化脚本留够认证时间，避免偶发超时                                    |


### 9.3 创建自动化运维专用账号

```bash
# 创建 DataX / Hive 等组件调用的专用账号
useradd -m -s /bin/bash ops

# 配置 SSH 密钥
mkdir -p /home/ops/.ssh
chmod 700 /home/ops/.ssh
# 将自动化任务发起方的公钥写入
echo "ssh-rsa AAAAB3..." >> /home/ops/.ssh/authorized_keys
chmod 600 /home/ops/.ssh/authorized_keys
chown -R ops:ops /home/ops/.ssh

# 按需配置免密 sudo（仅授权具体命令，遵循最小权限原则）
cat > /etc/sudoers.d/ops << 'EOF'
# ops 账号 sudo 权限——仅授权业务所需命令
ops ALL=(ALL) NOPASSWD: /usr/bin/datax
ops ALL=(ALL) NOPASSWD: /usr/bin/hive
ops ALL=(ALL) NOPASSWD: /usr/bin/hdfs
ops ALL=(ALL) NOPASSWD: /usr/bin/beeline
ops ALL=(ALL) NOPASSWD: /usr/bin/sqoop
EOF
chmod 440 /etc/sudoers.d/ops

# 验证
su - ops -c "sudo datax --version"
```

---

## 10. SELinux 配置

**目的：** 生产环境建议设置为 `permissive`（记录不拦截），而非完全关闭，保留审计能力。

```bash
# 临时设置（立即生效，无需重启）
setenforce 0

# 持久化设置
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# 验证
getenforce        # 应显示: Permissive
sestatus          # 确认配置已持久化
```

> **注意：** 如果业务依赖 Docker 或特殊目录访问，完全关闭 SELinux (`disabled`) 也可接受，但会丧失安全审计能力。

---

## 11. 主机防火墙基线

**目的：** 统一主机防火墙策略，避免每台服务器、每个组件文档分别维护 `firewalld` 规则，降低部署和排障复杂度。

本文默认服务器处于受控内网环境，业务访问控制统一由外层防火墙、安全组、ACL、VPN、堡垒机或统一网络策略承载。标准服务器默认关闭 `firewalld`，各业务组件部署文档不再维护本机防火墙放行规则。

> **重要说明：** 关闭 `firewalld` 不等于取消安全控制，而是将访问控制统一上收到外层网络边界。服务器端口不得直接暴露到公网；端口来源、用途和开放范围必须由外层网络策略统一管理并记录。

执行前确认当前网络边界和服务状态：

```bash
systemctl is-active firewalld 2>/dev/null || true
firewall-cmd --state 2>/dev/null || true
iptables -S 2>/dev/null | head -50
ss -tulnp
```

关闭并禁止 `firewalld`：

```bash
systemctl stop firewalld 2>/dev/null || true
systemctl disable firewalld 2>/dev/null || true
systemctl mask firewalld 2>/dev/null || true
```

验证：

```bash
systemctl is-enabled firewalld 2>/dev/null || true
systemctl is-active firewalld 2>/dev/null || true
systemctl status firewalld --no-pager 2>/dev/null || true
```

要求：

- 服务器端口不得直接暴露到公网；
- 端口访问来源必须由外层防火墙、安全组、ACL、VPN、堡垒机或统一网络策略限制；
- 访问策略变更应记录到资产台账、CMDB 或网络策略记录；
- 公网入口、边界代理、堡垒机、多租户主机、Docker/Kubernetes 节点或合规要求主机可作为例外单独启用主机防火墙；
- 例外主机的防火墙规则应由专项文档或自动化配置统一维护，不在单个业务组件部署文档中分散维护。

**Docker/Kubernetes 主机例外说明：**

Docker 和 Kubernetes 会操作 `iptables`/`nftables` 规则。容器主机是否关闭 `firewalld` 必须结合容器网络方案评估，不要手工清空 `iptables`，也不要在不了解 `DOCKER-USER`、CNI、Service、Ingress 规则的情况下重载防火墙。

---

## 12. fail2ban 防暴力破解

**目的：** 自动封禁多次 SSH 登录失败的 IP。

```bash
# 安装 EPEL 源和 fail2ban
# 本文默认 firewalld 已关闭，fail2ban 使用 iptables 动作，不安装 fail2ban-firewalld
yum install -y epel-release
yum install -y fail2ban

# 配置 fail2ban
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
# 封禁时间 1 小时
bantime = 3600
# 检测时间窗口 10 分钟
findtime = 600
# 最大失败次数
maxretry = 5
# 使用 iptables 动作。本文默认 firewalld 已按第 11 节关闭
banaction = iptables-multiport

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 7200
EOF

# 启动
systemctl enable fail2ban --now

# 验证
fail2ban-client status sshd
```

---

## 13. /tmp 分区加固

**目的：** 限制 /tmp 分区权限，防止恶意脚本执行。

```bash
# 方案A：将 /tmp 挂载为独立 tmpfs（推荐，重启自动清理）
cat >> /etc/fstab << 'EOF'
tmpfs   /tmp    tmpfs   defaults,noexec,nosuid,nodev,size=2G    0 0
EOF

# 方案B：如果 /tmp 在根分区上，通过 mount --bind 方式加固
# mkdir -p /tmp_new
# mount --bind /tmp_new /tmp
# mount -o remount,noexec,nosuid,nodev /tmp

# 重新挂载
mount -o remount /tmp

# 验证
mount | grep /tmp
# 应显示: noexec,nosuid,nodev
```

> **注意：** 某些应用（如 Docker、pip）可能需要在 /tmp 下执行文件，如果服务启动失败，检查是否与此配置冲突。

---

## 14. auditd 审计日志

**目的：** 记录关键系统操作（文件修改、用户切换、权限变更），满足等保要求。

```bash
# 安装 audit
yum install -y audit

# 配置审计规则
cat >> /etc/audit/rules.d/audit.rules << 'EOF'

# ================================================================
# 生产环境审计规则 - Author: 王有政
# ================================================================

# 监控用户/组变更
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/gshadow -p wa -k identity

# 监控 SSH 配置变更
-w /etc/ssh/sshd_config -p wa -k sshd_config

# 监控 sudo 操作
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# 监控 crontab 变更
-w /etc/crontab -p wa -k crontab
-w /etc/cron.d/ -p wa -k cron
-w /var/spool/cron/ -p wa -k cron

# 监控内核模块加载
-w /sbin/insmod -p x -k modules
-w /sbin/modprobe -p x -k modules

# 监控系统启动脚本
-w /etc/rc.d/ -p wa -k boot_scripts

# 监控日志删除操作
-w /var/log/ -p wa -k log_tampering

# 登录/注销事件
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins
EOF

# 重启 auditd
systemctl enable auditd --now
service auditd restart

# 查看审计日志
# ausearch -k identity --start today
# aureport --login
```

---

## 15. Swap 与内存管理

**目的：** 数据库/中间件服务器建议关闭 Swap 或降低 swappiness，避免内存换页导致性能抖动。

```bash
# 查看当前 Swap 状态
swapon --show
cat /proc/swaps

# 方案A：完全关闭 Swap（适合数据库服务器，内存 >= 16G，且必须在维护窗口执行）
free -h
swapon --show
# swapoff -a
# 只注释第三列为 swap 的挂载项，不要用 sed -i '/swap/d' 误删包含 swap 字符串的其他行
# cp -a /etc/fstab /etc/fstab.$(date +%F_%H%M%S).bak
# awk '($3=="swap"){$0="#"$0} {print}' /etc/fstab > /etc/fstab.new && mv /etc/fstab.new /etc/fstab
# 注释掉 /etc/fstab 中 swap 相关行

# 方案B：降低 swappiness（适合内存不充裕的应用服务器）
# 将 swappiness 设置为 1（尽量不使用 swap）
echo 'vm.swappiness = 1' >> /etc/sysctl.conf
sysctl -p

# 查看当前 swappiness
cat /proc/sys/vm/swappiness
```

**建议策略：**


| 服务器类型            | Swap 策略                 |
| ---------------- | ----------------------- |
| 数据库（MySQL/Redis） | 完全关闭 Swap               |
| 应用服务器            | swappiness = 1          |
| 内存 < 8G          | 保留 Swap，swappiness = 10 |


---

## 16. Transparent Huge Pages (THP) 配置

**目的：** THP 会导致数据库（MySQL、Redis、MongoDB）出现内存延迟抖动，生产环境必须关闭。

```bash
# 查看当前 THP 状态
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag

# 临时关闭
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 持久化关闭（创建 systemd 服务）
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=docker.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF

systemctl daemon-reload
systemctl enable disable-thp --now

# 验证
cat /sys/kernel/mm/transparent_hugepage/enabled
# 应显示: always madvise [never]
```

> **注意：** 如果服务器只跑 Java 应用（如 Spring Boot），THP 可以保留。但如果部署了 Redis、MySQL、MongoDB，必须关闭。

---

## 17. 挂载参数优化

**目的：** 优化文件系统挂载参数，减少不必要的磁盘写入。**应在 I/O 调度器之前完成，因为挂载方式影响后续 I/O 行为。**

```bash
# 查看当前挂载
mount | grep -E "^/dev"

# 编辑 /etc/fstab，为数据盘添加优化参数
# 关键参数说明:
#   noatime    - 不更新文件访问时间（减少写操作）
#   nodiratime - 不更新目录访问时间
#   defaults   - 默认参数

# 示例 /etc/fstab 配置:
# /dev/sdb1  /data  xfs  defaults,noatime,nodiratime  0 0

# 修改后重新挂载
# mount -o remount /data

# 验证
mount | grep /data
```

> **注意：** 修改 `/etc/fstab` 前务必备份原文件，错误配置可能导致系统无法启动。

---

## 18. 磁盘 I/O 调度器配置

**目的：** 根据磁盘类型选择最优调度器，提升 I/O 性能。

```bash
# 查看当前调度器
cat /sys/block/sda/queue/scheduler

# 查看磁盘类型（旋转=HDD，非旋转=SSD）
cat /sys/block/sda/queue/rotational
# 0 = SSD，1 = HDD
```

**调度器选择：**


| 磁盘类型 | 推荐调度器           | 说明               |
| ---- | --------------- | ---------------- |
| SSD  | noop / deadline | SSD 无寻道时间，简单调度即可 |
| HDD  | deadline        | 减少请求饥饿，适合数据库场景   |


```bash
# 临时设置 SSD 使用 deadline
echo deadline > /sys/block/sda/queue/scheduler

# 持久化设置（通过 udev 规则）
cat > /etc/udev/rules.d/60-io-scheduler.rules << 'EOF'
# SSD 使用 deadline
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="deadline"
# HDD 使用 deadline
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="deadline"
EOF

# 验证
udevadm control --reload-rules
udevadm trigger
cat /sys/block/sda/queue/scheduler
```

---

## 19. 内核参数优化 (sysctl)

**目的：** 优化网络、内存、文件系统等内核参数，提升系统整体性能。

```bash
# 备份原配置
cp /etc/sysctl.conf /etc/sysctl.conf.bak

# 追加优化参数
cat >> /etc/sysctl.conf << 'EOF'

# ================================================================
# 生产环境内核参数优化 - Author: 王有政
# ================================================================

# ---- 网络优化 ----
# 最大套接字发送/接收缓冲区 (16MB)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
# TCP 自动调优缓冲区 (min, default, max)
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
# 网络设备接收队列长度
net.core.netdev_max_backlog = 65536
# TCP accept 队列长度，Redis/MySQL/Nginx 等高并发服务依赖该参数
net.core.somaxconn = 65535
# SYN 队列长度
net.ipv4.tcp_max_syn_backlog = 65536
# 启用 TCP BBR 拥塞控制需内核支持。CentOS 7.9 默认 3.10 内核通常不支持 BBR，默认保留 cubic
# 执行前先检查: sysctl net.ipv4.tcp_available_congestion_control
# net.core.default_qdisc = fq
# net.ipv4.tcp_congestion_control = bbr
# TIME-WAIT 复用
net.ipv4.tcp_tw_reuse = 1
# 最大 TIME-WAIT 数量
net.ipv4.tcp_max_tw_buckets = 20000
# 端口范围扩展
net.ipv4.ip_local_port_range = 1024 65535
# TCP 保活探测（10 分钟无活动后探测，30 秒间隔，10 次失败断开）
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
# 防 SYN 洪水攻击
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_synack_retries = 2
# FIN 超时缩短
net.ipv4.tcp_fin_timeout = 15
# 连接跟踪最大值（高并发场景必须调整）
net.netfilter.nf_conntrack_max = 1048576

# ---- 文件系统 ----
# 系统允许的最大打开文件数
fs.file-max = 1048576
# inotify 监控上限（Docker/Java 应用常用）
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
# 允许的最大进程 ID
kernel.pid_max = 65536

# ---- 内存管理 ----
# Redis/MySQL 等数据库后台 fork、RDB/AOF rewrite 依赖该参数，避免内存分配策略导致 fork 失败
vm.overcommit_memory = 1
# 脏页占比控制，避免瞬间 IO 尖刺
vm.dirty_ratio = 20
vm.dirty_background_ratio = 5
# 内存不足时的 OOM 策略
vm.panic_on_oom = 0
# ARP 配置（集群环境下重要）
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2

# ---- 安全加固 ----
# 禁止 ICMP 重定向（防中间人攻击）
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
# 多网卡、VPN、专线、策略路由、LVS、容器网络环境不建议全局强制 rp_filter=1，可按需改为 2 或按接口单独配置
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
# 普通非容器主机可禁止 IP 转发；Docker/K8s/网关主机必须设为 1
# net.ipv4.ip_forward = 0
net.ipv4.ip_forward = 1
# 禁用 IPv6（如业务不使用）
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
# 内核地址空间随机化
kernel.randomize_va_space = 2
# 限制 dmesg 访问
kernel.dmesg_restrict = 1
EOF

# 加载生效
sysctl -p

# 验证关键参数
sysctl net.ipv4.tcp_congestion_control
sysctl fs.file-max
sysctl vm.overcommit_memory
sysctl net.core.somaxconn
sysctl vm.dirty_ratio
```

> **注意：** 如果 CentOS 7.9 内核不支持 BBR（`tcp_congestion_control` 报错），将 BBR 两行注释掉即可，不影响其他参数。Docker 环境必须确保 `net.ipv4.ip_forward = 1`，请按需调整。

---

## 20. 文件句柄数限制

**目的：** 高并发场景下默认的 1024 文件描述符远远不够，必须调整。

```bash
# 查看当前限制
ulimit -n   # 文件描述符
ulimit -u   # 最大进程数
ulimit -l   # 锁定内存

# 追加 limits 配置
cat >> /etc/security/limits.conf << 'EOF'

# ================================================================
# 生产环境句柄数配置 - Author: 王有政
# ================================================================
# soft: 软限制，可由用户自行调整到 hard 值
# hard: 硬限制，需 root 权限调整
# nofile: 最大打开文件描述符数
# nproc: 最大进程数
# memlock: 锁定内存大小（数据库/大页场景需要）

*       soft    nofile    1048576
*       hard    nofile    1048576
*       soft    nproc     65536
*       hard    nproc     65536
*       soft    memlock   unlimited
*       hard    memlock   unlimited

# root 用户单独配置（避免与其他用户冲突）
root    soft    nofile    1048576
root    hard    nofile    1048576
root    soft    nproc     65536
root    hard    nproc     65536
EOF

# 同时修改 systemd 的默认限制（Docker/systemd 管理的服务需要此配置）
cat >> /etc/systemd/system.conf << 'EOF'
DefaultLimitNOFILE=1048576
DefaultLimitNPROC=65536
DefaultLimitMEMLOCK=infinity
EOF

cat >> /etc/systemd/user.conf << 'EOF'
DefaultLimitNOFILE=1048576
DefaultLimitNPROC=65536
EOF

# 重新加载 systemd
systemctl daemon-reload

# 需要重新登录 shell 才能生效，验证:
# ulimit -n  -> 1048576
# ulimit -u  -> 65536
```

---

## 21. 网络接口优化

**目的：** 优化网卡缓冲区和队列长度，提升网络吞吐。

```bash
# 查看当前网卡名称
ip link show

# 查看网卡 ring buffer
ethtool -g eth0

# 设置网卡发送队列长度（默认 1000，高并发可调到 5000）
ip link set eth0 txqueuelen 5000

# 持久化（通过 /etc/rc.local 或 NetworkManager dispatcher）
cat > /etc/rc.d/rc.local << 'EOF'
#!/bin/bash
# 网卡优化
ip link set eth0 txqueuelen 5000
EOF
chmod +x /etc/rc.d/rc.local

# 查看网卡 ring buffer 可设置范围
ethtool -g eth0
# 如果需要调大（示例）：
# ethtool -G eth0 rx 4096 tx 4096

# 验证
ip -s link show eth0
```

---

## 22. SysRq 紧急调试开关

**目的：** 系统完全卡死时，可通过 SysRq 组合键安全重启。

```bash
# 启用 SysRq
echo "kernel.sysrq = 1" >> /etc/sysctl.conf

# 生效
sysctl -p

# 验证
cat /proc/sys/kernel/sysrq   # 应显示 1
```

**常用 SysRq 组合键（通过 Alt+SysRq+键 或 echo 到 /proc/sysrq-trigger）：**


| 操作    | 命令                             | 说明           |
| ----- | ------------------------------ | ------------ |
| 同步磁盘  | `echo s > /proc/sysrq-trigger` | 刷新所有脏页到磁盘    |
| 只读重挂载 | `echo u > /proc/sysrq-trigger` | 将文件系统重新挂载为只读 |
| 安全重启  | `echo b > /proc/sysrq-trigger` | 不同步直接重启（慎用）  |
| 安全关机  | `echo o > /proc/sysrq-trigger` | 安全关机         |


> **安全重启三步走：** `echo s` → `echo u` → `echo b`

---

## 23. JDK 安装与环境变量设置

**目的：** 安装生产 Java 运行环境，并统一配置全局环境变量。本文以离线安装 `OpenJDK21U-jdk_x64_linux_hotspot_21.0.11_10.tar.gz` 为例，安装目录遵循 `/data/module` 规范。

> **版本选择建议：** 新项目优先使用 JDK 21 LTS；存量 Spring Boot 2.x、老中间件或历史应用如果只验证过 JDK 8/11/17，不要直接升级到 JDK 21，应先在测试环境完成兼容性验证。

```bash
# 进入软件包所在目录
cd /data/module

# 确认安装包存在
ls -lh OpenJDK21U-jdk_x64_linux_hotspot_21.0.11_10.tar.gz

# 解压 JDK
# 解压后通常生成 jdk-21.0.11+10 目录
tar -zxvf OpenJDK21U-jdk_x64_linux_hotspot_21.0.11_10.tar.gz

# 确认实际解压目录
ls -ld jdk-21.0.11+10

# 建立稳定软链接，后续升级只需要切换软链接
ln -sfn /data/module/jdk-21.0.11+10 /data/module/jdk21

# root 拥有安装目录，普通业务用户只读执行
chown -R root:root /data/module/jdk-21.0.11+10
chmod -R go-w /data/module/jdk-21.0.11+10
```

配置全局环境变量：

> **统一约定：** 团队自定义环境变量统一维护在 `/etc/profile.d/myenv.sh`，不要为 JDK 单独创建 `/etc/profile.d/jdk.sh`。如果该文件已包含 MySQL、Maven、Redis 等环境变量，禁止使用 `cat >` 覆盖，应编辑文件追加 JDK 配置。

编辑统一环境变量文件：

```bash
vim /etc/profile.d/myenv.sh
```

追加以下内容：

```bash
# JAVA_HOME：JDK 当前版本软链接目录，统一指向 /data/module/jdk21，升级时只切换软链接
export JAVA_HOME=/data/module/jdk21
export JRE_HOME=$JAVA_HOME
export CLASSPATH=.:$JAVA_HOME/lib

# PATH：将 Java 命令加入命令搜索路径，便于直接执行 java、javac、jcmd、jmap 等命令
export PATH=$JAVA_HOME/bin:$PATH
```

保存后加载生效：

```bash
chmod 644 /etc/profile.d/myenv.sh
source /etc/profile.d/myenv.sh

# 验证环境变量和 Java 版本
echo $JAVA_HOME
which java
java -version
javac -version
```

**验收标准：**

| 项目 | 期望结果 |
| --- | --- |
| `echo $JAVA_HOME` | `/data/module/jdk21` |
| `which java` | `/data/module/jdk21/bin/java` |
| `java -version` | 显示 `openjdk version "21.0.11"` |
| `javac -version` | 显示 `javac 21.0.11` |

> **升级建议：** 后续升级 JDK 时，不要覆盖原目录。先解压新版本目录，再用 `ln -sfn /data/module/<新JDK目录> /data/module/jdk21` 切换软链接，验证通过后再安排旧版本清理。


---

## 24. 常用诊断工具安装

**目的：** 生产故障排查必备工具（基础工具已在第 3 节安装）。**Docker 安装前应确保诊断工具就绪，便于定位安装问题。**

```bash
# 网络诊断
yum install -y \
    bind-utils \
    tcpdump \
    telnet \
    nload \
    iftop

# 系统诊断与压测
yum install -y \
    lsof \
    strace \
    htop \
    iotop \
    sysstat \
    jq \
    perf \
    numactl

# 启动 sysstat 定时采集（每 10 秒采样）
systemctl enable sysstat --now

# 验证
sar -u 1 3    # CPU 使用率
sar -r 1 3    # 内存使用率
iostat -x 1 3 # 磁盘 I/O
```

---

## 25. Docker 安装与配置

**目的：** 安装 Docker 并配置生产级参数。

```bash
# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 Docker 官方源
# 前文已经配置 Docker CE 源时，不要重复添加官方公网源；内网环境应使用企业 yum 源或离线 RPM 包
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 查看仓库中可用版本，再选择经过测试验证的版本
yum list docker-ce --showduplicates | sort -r | head -20

# 安装指定版本（推荐安装稳定版本，而非最新版）
yum install -y docker-ce-24.0.7 docker-ce-cli-24.0.7 containerd.io

# 创建 Docker 数据目录（务必放在独立数据盘 /data）
mkdir -p /data/docker

# Docker 前置检查：overlay2 建议使用 xfs ftype=1 或 ext4；网段、DNS、镜像仓库需先确认
findmnt /data
xfs_info /data 2>/dev/null | grep ftype || true
ip route
cat /etc/resolv.conf

# Docker 生产级配置
cat > /etc/docker/daemon.json << 'EOF'
{
  "data-root": "/data/docker",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "insecure-registries": [],
  "live-restore": true,
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 5,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 1048576,
      "Soft": 1048576
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "bip": "172.30.255.1/24",
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    }
  ]
}
EOF

# 启动 Docker
systemctl daemon-reload
systemctl enable docker --now

# 验证
docker info
docker version
docker run --rm hello-world

# 清理测试镜像
docker rmi hello-world
```

**daemon.json 关键配置说明：**

> **实战避坑：** `daemon.json` 中不要默认写死公网 DNS。企业内网域名、云厂商私有域名、Nacos/Consul/Harbor 等经常依赖内部 DNS，写死 `8.8.8.8` 可能导致容器内解析内网服务失败。默认让 Docker 继承宿主机 `/etc/resolv.conf`，如必须配置 DNS，应使用企业 DNS 或云厂商内网 DNS。


| 参数                            | 说明                         |
| ----------------------------- | -------------------------- |
| `data-root`                   | Docker 数据存储位置，必须在独立数据盘     |
| `log-driver + log-opts`       | 限制容器日志大小，防止撑爆磁盘            |
| `live-restore`                | Docker daemon 重启时不影响运行中的容器 |
| `native.cgroupdriver=systemd` | 与 systemd 统一 cgroup 管理     |
| `default-ulimits`             | 容器默认句柄数限制                  |
| `max-concurrent-downloads`    | 并发拉取镜像数，避免占用过多带宽           |
| `bip`                         | 默认 `bridge/docker0` 网桥地址段，需避开企业内网/VPC/VPN/专线网段 |
| `default-address-pools`       | Docker 自定义 bridge 网络地址池，需统一规划，避免与内网网段冲突 |


---

## 26. 日志配置

**目的：** 防止日志撑爆磁盘，确保日志可追溯。

```bash
# ---- journald 日志限制 ----
cat > /etc/systemd/journald.conf << 'EOF'
[Journal]
Storage=persistent
SystemMaxUse=2G
SystemMaxFileSize=200M
MaxRetentionSec=30day
ForwardToSyslog=yes
EOF

systemctl restart systemd-journald

# ---- 应用日志轮转 ----
cat > /etc/logrotate.d/custom_apps << 'EOF'
/data/apps/*/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d
    size 500M
}
EOF

# ---- Docker 容器日志轮转 ----
# 已在 Docker daemon.json 中配置（见 Docker 章节）

# 手动测试 logrotate
logrotate -d /etc/logrotate.d/custom_apps

# 验证 journald
journalctl --disk-usage
```

---

## 27. Core Dump 配置

**目的：** 应用崩溃时生成 core dump 文件，用于离线分析故障原因。

```bash
# 创建 core dump 存储目录
mkdir -p /data/coredump
chmod 777 /data/coredump

# 配置 core dump 文件路径和命名规则
# %p=进程PID, %t=时间戳, %e=程序名
echo "kernel.core_pattern = /data/coredump/core.%e.%p.%t" >> /etc/sysctl.conf

# 不限制 core dump 文件大小
echo "*    soft    core    unlimited" >> /etc/security/limits.conf
echo "*    hard    core    unlimited" >> /etc/security/limits.conf

# 生效
sysctl -p

# 验证
cat /proc/sys/kernel/core_pattern
ulimit -c   # 应显示: unlimited
```

> **注意：** Core dump 文件可能很大（与进程内存相当），确保 `/data/coredump` 有足够空间，并配置定时清理策略。

---

## 28. 最终验证清单

**完成所有步骤后，逐项检查：**

```bash
echo "=============================="
echo " 生产环境初始化验证清单"
echo "=============================="

echo ""
echo "--- 1. 系统信息 ---"
hostname
cat /etc/redhat-release
uname -r
free -h
df -h / /data

echo ""
echo "--- 2. 时区/NTP ---"
timedatectl | grep -E "Time zone|NTP"
chronyc tracking | grep "Leap status"

echo ""
echo "--- 3. 安全状态 ---"
echo "SELinux: $(getenforce)"
echo "firewalld: $(systemctl is-active firewalld 2>/dev/null || echo '已关闭')"

echo ""
echo "--- 4. SSH ---"
grep -E "^Port|^PermitRootLogin|^PasswordAuthentication|^MaxSessions|^MaxStartups|^ClientAliveCountMax|^TCPKeepAlive" /etc/ssh/sshd_config

echo ""
echo "--- 5. fail2ban ---"
systemctl is-active fail2ban 2>/dev/null || echo "fail2ban 未安装"

echo ""
echo "--- 6. auditd ---"
systemctl is-active auditd

echo ""
echo "--- 7. 内存管理 ---"
swapon --show || echo "Swap 已关闭"
cat /proc/sys/vm/swappiness
cat /sys/kernel/mm/transparent_hugepage/enabled

echo ""
echo "--- 8. 句柄数 ---"
ulimit -n
ulimit -u

echo ""
echo "--- 9. 内核参数 ---"
sysctl net.ipv4.tcp_congestion_control
sysctl fs.file-max
sysctl vm.dirty_ratio

echo ""
echo "--- 10. Docker ---"
systemctl is-active docker
docker info 2>/dev/null | grep -E "Docker Root Dir|Storage Driver|Logging Driver"

echo ""
echo "--- 11. Core Dump ---"
cat /proc/sys/kernel/core_pattern
ulimit -c

echo ""
echo "=============================="
echo " 验证完成！请重启服务器使所有配置生效。"
echo "=============================="
```

---

## 附录：配置文件变更清单


| 文件                                        | 变更内容               |
| ----------------------------------------- | ------------------ |
| `/etc/selinux/config`                     | SELinux=permissive |
| `/etc/sysctl.conf`                        | 内核参数优化（网络/内存/安全）   |
| `/etc/security/limits.conf`               | 句柄数/进程数/锁定内存       |
| `/etc/systemd/system.conf`                | systemd 默认限制       |
| `/etc/ssh/sshd_config`                    | SSH 安全加固           |
| `/etc/profile.d/custom.sh`                | 环境变量与别名            |
| `/etc/chrony.conf`                        | NTP 时间同步           |
| `/etc/systemd/journald.conf`              | 日志限制               |
| `/etc/logrotate.d/custom_apps`            | 应用日志轮转             |
| `/etc/docker/daemon.json`                 | Docker 生产配置        |
| `/etc/fstab`                              | 挂载参数优化、/tmp 加固     |
| `/etc/hosts`                              | 主机名映射              |
| `/etc/login.defs`                         | 密码策略               |
| `/etc/security/pwquality.conf`            | 密码复杂度              |
| `/etc/audit/rules.d/audit.rules`          | 审计规则               |
| `/etc/fail2ban/jail.local`                | fail2ban 规则        |
| `/etc/udev/rules.d/60-io-scheduler.rules` | I/O 调度器            |
| `/etc/systemd/system/disable-thp.service` | THP 关闭服务           |


> **Author:** 王有政 | **最后更新:** 2026-06-15

