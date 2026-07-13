# TDengine 3.4 OSS 单体生产级配置部署手册

> **Author:** 王有政<br>
> **日期:** 2026-07-13<br>
> **适用系统:** CentOS 7.9 x86_64<br>
> **候选版本:** TDengine TSDB-OSS 3.4.1.13<br>
> **文档定位:** 单体 TDengine 3.4 OSS 的容量评估、Linux-Generic 离线安装、配置、安全、备份恢复、监控、验证、升级与回滚。集群、高可用、跨机房容灾和 Enterprise 专属能力另行成册。

---

## 目录

1. [适用范围与结论先行](#1-适用范围与结论先行)
2. [官方核验结论与版本决策](#2-官方核验结论与版本决策)
3. [部署前检查](#3-部署前检查)
4. [安装方案对比](#4-安装方案对比)
5. [容量规划](#5-容量规划)
6. [目录、账号与权限](#6-目录账号与权限)
7. [制品准备与校验](#7-制品准备与校验)
8. [安装 TDengine](#8-安装-tdengine)
9. [端口与网络访问控制](#9-端口与网络访问控制)
10. [生产配置](#10-生产配置)
11. [启动、初始化与改密](#11-启动初始化与改密)
12. [备份与恢复](#12-备份与恢复)
13. [监控与告警](#13-监控与告警)
14. [生产验证清单](#14-生产验证清单)
15. [日常运维](#15-日常运维)
16. [升级与回滚](#16-升级与回滚)
17. [风险与限制](#17-风险与限制)
18. [参考资料](#18-参考资料)

---

## 1. 适用范围与结论先行

本文适用于以下场景：

- CentOS 7.9 x86_64 存量服务器；
- 单节点 TDengine TSDB-OSS 3.4；
- 需要离线制品审计、自定义程序目录、systemd 托管、备份恢复和监控；
- 业务可以接受单节点计划停机和单点故障风险。

本文不覆盖：

- 多 dnode 集群、mnode 高可用和多副本；
- 滚动升级；
- 跨机房容灾、实时同步和自动故障切换；
- Enterprise 增量备份、时间点恢复、审计、IP 白名单、数据加密等专属能力；
- taosX、数据迁移平台和 Explorer 对公网发布。

生产部署结论：

1. 安装主流程推荐使用 **官方 Linux-Generic x64 压缩包 + 内网制品库 + 安装脚本审计 + 自定义版本目录**；
2. 候选版本固定为 `3.4.1.13`，但必须先通过预生产兼容性、备份恢复和压测验收；
3. `taosd`、`taosAdapter` 和 `taosKeeper` 按需启动，`taos-explorer` 默认关闭；
4. 保留官方端口 `6030`、`6041`、`6043`，通过来源白名单实现边界隔离；
5. 数据目录使用独立 SSD/NVMe 数据盘，备份必须复制到另一台主机或独立存储；
6. OSS 主备份手段是 `taosdump` 逻辑全量/分段备份，不具备 Enterprise 的持续增量和时间点恢复能力；
7. 单体实例不是高可用方案，不得承诺自动故障切换或零停机升级。

---

## 2. 官方核验结论与版本决策

### 2.1 2026-07-13 官方核验结果

| 核验项 | 官方结果 | 本文处理 |
|---|---|---|
| 候选版本 | 官方下载中心存在 `3.4.1.13` | 可作为候选制品 |
| RPM 文件 | `tdengine-tsdb-oss-3.4.1.13-linux-x64.rpm` | 备选安装制品，仍保留校验说明 |
| RPM 大小 | 官网显示 `132.75MB` | 仅作人工复核，不替代哈希 |
| RPM MD5 | `e1ad92cbdcc3d459243e3311e7b259e4` | 与官网下载页交叉核验 |
| Generic 文件 | `tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz` | **本文安装主制品** |
| Generic 大小 | `300274656` 字节，官网显示 `286.36MB` | HTTP HEAD 与下载中心显示一致 |
| Generic MD5 | `4c0fc6e31d59107a5125d5284fc58c32` | 与官网下载页交叉核验 |
| Generic SHA256 | `1b8cba22f1ea510ec72863041413ad86970ece2501d15c6eff482db189add515` | 本次下载实算值，非官网公布值 |
| Generic 脚本 | `install.sh` 1647 行，支持 `-d` 自定义安装父目录 | 已按 3.4.1.13 实包审计 |
| CentOS 7 非 root 安装 | 脚本明确拒绝 systemd 219 的非 root 安装 | 本文使用 root 安装并登记风险 |
| 系统支持 | 3.4.1 文档列出 CentOS 7.9+、x64 已测试 | 满足候选环境 |
| 当前文档分支 | 官网提供 3.4.1 文档；当前 Latest 为 3.4.2 | 命令以 3.4.1 文档为准 |
| LTS 状态 | 官网版本选择器标注 3.3.6 为 LTS，未标注 3.4.1 为 LTS | 3.4.1.13 不按 LTS 宣传 |
| 发布说明 | 下载中心的 3.4.1.13 Release Notes 链接当前返回 404 | 上线前必须向供应方/社区补齐变更与缺陷信息 |
| 默认配置 | `/etc/taos/taos.cfg`，数据 `/var/lib/taos`，日志 `/var/log/taos` | 配置文件保留，数据和日志迁往独立盘 |
| 核心端口 | Native `6030/TCP`、REST/WebSocket `6041/TCP`、Keeper `6043/TCP` | 保留默认端口并限制来源 |
| 备份能力 | OSS 使用 `taosdump`，备份运行中的实例，格式为 Apache Avro | 主备份方案 |
| 监控能力 | taosKeeper 提供 `/metrics/v2`，官方推荐该端点 | Prometheus 采集入口 |

> 官网显示的文件大小可能因显示单位和页面更新产生差异，不能作为完整性校验依据。MD5 只能用于发现传输损坏，不能单独证明制品来源可信。

### 2.2 版本选择方案

| 方案 | 优点 | 风险 | 结论 |
|---|---|---|---|
| 方案 A：`3.4.1.13` | 与当前候选一致；官方仍提供 OSS Generic/RPM；维护版本号较高 | 非官网标注 LTS；发布说明链接失效；CentOS 7 已 EOL | **推荐，但必须通过验收门禁** |
| 方案 B：`3.3.6.13` LTS | 官网明确标注 3.3.6 为 LTS；稳定性定位更清晰 | 可能缺少 3.4 功能；需要重新做功能兼容确认 | 稳定优先时评估 |
| 方案 C：当前 3.4.2 系列 | 当前文档 Latest；可获得后续修复 | 不是本项目已选候选；应用驱动和数据兼容需重新验证 | 后续升级候选 |

**推荐方案 A 的上线门禁：**

- 官方 Generic 包、MD5、企业 SHA256 清单均校验通过，`install.sh` 审计记录已归档；
- 预生产连续运行不少于 7 天；
- 完成真实数据模型写入、查询、保留策略和压力测试；
- 完成一次异机或隔离库恢复演练；
- 确认 Java/Python/Go 等连接器与服务端兼容；
- 获取并评审 3.4.1.13 的变更、已知缺陷或社区发布记录；
- 明确 CentOS 7 的网络隔离、补丁补偿和迁移计划。

---

## 3. 部署前检查

### 3.1 服务器最低基线

| 资源 | 最低起步 | 通用生产建议 | 说明 |
|---|---:|---:|---|
| CPU | 8 核 | 16 核及以上 | 以真实写入和查询压测校准 |
| 内存 | 16 GB | 64 GB 及以上 | 同时考虑 vnode、查询和系统页缓存 |
| 系统盘 | 50 GB | 100 GB SSD | 不承载时序数据 |
| 数据盘 | 200 GB SSD | 独立 NVMe/企业级 SSD | 按第 5 节计算并预留扩容空间 |
| 备份空间 | 1 个全量 | 2 个全量以上且异机 | 不得与唯一数据副本共盘 |
| 网络 | 千兆内网 | 万兆内网 | 丢包率应低于万分之一 |
| 时间同步 | 必须 | chrony | 时序数据依赖准确时间 |

以上是部署起点，不是性能 SLA。官方指出写入批次、表数量、vgroup、查询模型和数据压缩率都会显著改变资源需求。

### 3.2 系统检查

```bash
cat /etc/redhat-release
uname -m
uname -r
hostnamectl
hostname -f
lscpu
free -h
lsblk -o NAME,SIZE,ROTA,TYPE,FSTYPE,MOUNTPOINT
df -hT
mount | grep ' /data '
timedatectl
chronyc tracking
getenforce
systemctl is-active firewalld 2>/dev/null || true
ulimit -n
```

验收要求：

- 系统必须为 CentOS 7.9 x86_64；
- `hostname -f` 返回稳定、唯一且可解析的名称；
- 数据盘已挂载，文件系统建议 XFS；
- 时间已同步，时区、locale、charset 上线后不得随意变更；
- 服务器初始化应先执行《CentOS7.9生产环境初始化手册.md》；
- THP、Swap、文件句柄和 sysctl 以初始化手册为准，不在本文重复覆盖。

### 3.3 FQDN 和地址解析

单体也必须固定 FQDN。示例：

```bash
hostnamectl set-hostname prod-tsdb-01

# 按现场 IP 修改；不要把业务 FQDN 解析到 127.0.0.1
echo '<TDENGINE_INNER_IP> prod-tsdb-01' >> /etc/hosts

getent hosts prod-tsdb-01
ping -c 3 prod-tsdb-01
```

`fqdn` 是 TDengine 节点身份的一部分。实例初始化后禁止直接修改主机名、FQDN 或数据目录并期待原数据自动适配。

### 3.4 历史实例排查

```bash
rpm -qa | grep -Ei 'tdengine|taos' || true
systemctl list-unit-files | grep -Ei 'taos' || true
ps -ef | grep -E '[t]aosd|[t]aosadapter|[t]aoskeeper' || true
ss -lntup | grep -E ':(6030|6041|6043|6044|6045|6046|6047|6048|6049|6060)\b' || true
find /etc/taos /var/lib/taos /var/log/taos -maxdepth 2 -ls 2>/dev/null
```

发现历史实例时，先确认业务、版本、数据目录、备份和停机窗口。禁止直接删除 `/var/lib/taos` 或使用新版本覆盖未知数据。

---

## 4. 安装方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| 方案 A：Linux-Generic + 审计 `install.sh` + `-d` 自定义目录 | 程序主体可落入带版本目录；脚本和文件可事前审计；便于保留旧版本 | 仍会创建系统级软链接、库链接、配置和 systemd unit；不是纯解压运行 | **本文推荐** |
| 方案 B：官方 RPM + 本地 Yum | 依赖、卸载和升级记录最规范 | 程序路径由包定义；不符合本项目对版本目录的偏好 | 大规模包管理环境 |
| 方案 C：官方 RPM + `rpm -ivh` | 与官网下载页命令一致；步骤短 | 缺失依赖时需手工处理 | 极简离线环境 |
| 方案 D：Docker 单机 | 环境封装、启动快 | 数据卷、网络、日志、升级和宿主机资源约束更复杂 | 已有成熟容器平台时 |

**推荐方案 A。** 对 3.4.1.13 实包审计确认，`install.sh` 支持：

```text
-v [server | client]    安装服务端或客户端
-e [yes | no]           是否交互设置 FQDN
-d [install dir]        自定义安装父目录；脚本会在其下追加 /taos
-s                      静默安装并关闭 FQDN 交互
```

本文使用 `-d /data/module/tdengine-3.4.1.13`，最终程序目录为：

```text
/data/module/tdengine-3.4.1.13/taos
```

必须准确理解其控制边界：Generic 包不是“解压后直接执行”。以 root 运行时，脚本还会写入或创建：

- `/usr/bin`：命令和服务程序软链接；
- `/usr/lib`、`/usr/lib64`：客户端动态库软链接；
- `/usr/include`：头文件软链接；
- `/etc/taos`：配置文件；
- `/etc/systemd/system`：`taosd`、`taosadapter`、`taoskeeper`、`taos-explorer` unit；
- `/var/lib/taos`、`/var/log/taos`：初始数据和日志目录；
- `/data/module/tdengine-3.4.1.13/taos/{cfg,data,log}`：指向上述系统目录的软链接。

因此本文所谓“可控”是指：制品、脚本、版本目录和系统级副作用全部经过审计并记录，而不是宣称零系统改动。

3.4.1.13 Linux-Generic x64 官方 CDN 直链：

[tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz](https://downloads.taosdata.com/tdengine-tsdb-oss/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz)

2026-07-13 使用 HTTP HEAD 复核结果为 `200 OK`，`Content-Length` 为 `300274656` 字节。生产服务器仍不建议直接访问公网，应由受控终端下载、校验并上传企业制品库。

---

## 5. 容量规划

### 5.1 必须收集的业务输入

上线前填写以下表格：

| 参数 | 符号 | 现场值 |
|---|---:|---:|
| 设备/子表数量 | `N` | `<填写>` |
| 每设备每秒记录数 | `R` | `<填写>` |
| 单行原始字节数 | `B` | `<填写>` |
| 在线保留天数 | `D` | `<填写>` |
| 峰值/平均写入倍数 | `Kpeak` | `<填写>` |
| 真实压缩比 | `C` | `<压测>` |
| 并发查询数 | `Q` | `<填写>` |
| 最大结果集 | `Result` | `<填写>` |
| 目标 RPO/RTO | - | `<填写>` |

### 5.2 存储计算

```text
日原始数据量 = N × R × B × 86400
保留期原始量 = 日原始数据量 × D
数据文件估算 = 保留期原始量 ÷ 实测压缩比 C
生产数据盘需求 = (数据文件估算 + WAL/元数据/临时空间) ÷ 0.70
```

官方资料指出多数场景压缩率通常不低于 5 倍，但压缩率高度依赖数据特征。生产容量不得直接把 `5` 当成保证值，必须用代表性 CSV 和表结构执行 `taosBenchmark`，写入后执行：

```sql
FLUSH DATABASE <数据库名>;
```

再在操作系统侧统计实际数据目录，不含 WAL 后计算压缩比：

```bash
du -sh /data/tdengine/data
du -hd1 /data/tdengine/data/vnode --exclude=wal 2>/dev/null
```

容量红线：

- 正常运行目标不超过数据盘 70%；
- 70% 告警，80% 严重告警；
- `minimalDataDirGB` 不是容量管理替代品，只是最后保护线；
- 备份、数据和临时目录不得在同一容量预算中重复计算；
- 保留策略必须在建库时通过 `KEEP` 明确，并由数据所有者签字确认。

### 5.3 内存计算

官方给出的数据库 vnode 内存估算公式为：

```text
vgroups × replica × (buffer + pages × pagesize + cachesize)
```

单体 OSS 应使用 `replica 1`。此外还要预留：

- taosd 查询、排序、聚合和连接内存；
- taosAdapter WebSocket/REST 连接内存；
- taosKeeper 和操作系统页缓存；
- 备份、压测和运维命令的瞬时内存。

官方指出 WebSocket 查询每个连接通常额外消耗约 8 MB；REST 返回大 JSON 时不能分批拉取，可能消耗更多内存。大结果集优先使用 WebSocket 流式读取，并在应用侧限制时间范围和行数。

### 5.4 CPU、vgroup 与写入批次

官方建议每个 CPU 核服务约 1～2 个 vnode，并建议常态 CPU 使用率保持在 50% 以下。单体方案的决策选项：

| 方案 | vgroup 策略 | 优点 | 风险 |
|---|---|---|---|
| 方案 A：数据库默认值 | 默认 2 个 vgroup | 简单，适合小规模起步 | 可能无法利用多核 |
| 方案 B：按实测规划 | 基于 CPU、写入并发和查询压测设置 | **推荐，可审计** | 需要压测 |
| 方案 C：按最大核数预配 | 接近每核 1～2 vnode | 初期并行度高 | vnode 内存和管理开销可能过大 |

**推荐方案 B。** 不在通用手册里硬编码 `vgroups`。先按真实数据模型测试，再通过建库参数固化。写入端应批量提交；官方指出单请求包含更多记录时效率显著高于逐行请求。

### 5.5 三档资源起步方案

| 方案 | CPU/内存 | 数据盘 | 适用说明 |
|---|---|---|---|
| 方案 A：低负载单体 | 8C/32G | 独立 SSD | 非关键、小规模；必须压测 |
| 方案 B：通用生产单体 | 16C/64G | 企业级 SSD/NVMe | **推荐起步规格** |
| 方案 C：高负载单体 | 32C/128G+ | 多块企业级 NVMe | 仍存在单点；高于此规模优先评估集群 |

规格只表示采购起点。最终容量必须以峰值写入、典型查询、备份窗口和恢复时间实测为准。

---

## 6. 目录、账号与权限

### 6.1 目录规划

| 路径 | 用途 | 权限建议 |
|---|---|---|
| `/data/module/tdengine-3.4.1.13/taos` | Generic 脚本安装的版本程序目录 | `root:root`，禁止业务写入 |
| `/data/module/tdengine` | 团队维护的当前版本软链接 | 指向上方版本目录，仅用于巡检和脚本引用 |
| `/etc/taos` | taosd、Adapter、Keeper 配置 | `root` 管理，含密码文件 `0640` |
| `/data/tdengine/data` | taosd 数据和 WAL | 运行用户可读写，`0750` |
| `/data/tdengine/log/taosd` | taosd 日志 | taosd 运行用户可读写 |
| `/data/tdengine/log/adapter` | taosAdapter 日志 | Adapter 运行用户可读写 |
| `/data/tdengine/log/keeper` | taosKeeper 日志 | Keeper 运行用户可读写 |
| `/data/tdengine/tmp` | TDengine 临时文件 | taosd 运行用户可读写，`0750` |
| `/backup/tdengine` | 本机备份暂存 | 独立盘优先，`0750` |

`/backup/tdengine` 若仍位于同一块物理盘，只是暂存目录，不算有效灾备副本。

### 6.2 运行账号决策

| 方案 | 说明 | 结论 |
|---|---|---|
| 方案 A：保持 Generic 包原生 systemd 运行方式 | 3.4.1.13 实包 unit 未设置 `User=`，即 root 运行 | **本文推荐，厂商兼容风险最低** |
| 方案 B：统一改为专用 `taos` 用户 | 权限隔离更强 | 必须经过完整回归，升级可能覆盖 unit |
| 方案 C：容器内非 root | 由容器镜像和编排约束 | 不属于本文主流程 |

安装后先识别实际服务账号：

```bash
for svc in taosd taosadapter taoskeeper; do
  echo "===== ${svc} ====="
  systemctl show "$svc" -p User -p Group -p FragmentPath 2>/dev/null || true
  systemctl cat "$svc" 2>/dev/null | grep -E '^(User|Group|ExecStart)=' || true
done
```

3.4.1.13 包内 `taosd.service`、`taosadapter.service`、`taoskeeper.service` 和 `taos-explorer.service` 均未设置 `User=`。此外，`install.sh` 明确判断 CentOS/RHEL 7 的 systemd 219 不支持其非 root 安装模式并直接退出。因此本文保留厂商 root 运行方式，同时通过网络边界、目录权限、停用 Explorer 和最小开放端口降低风险。改为专用账号必须另做专项验证，不能只修改 unit 的 `User=`。

### 6.3 创建目录并授权

执行 Generic 安装后、首次启动前执行：

```bash
TAOSD_USER="$(systemctl show taosd -p User --value)"
[ -n "$TAOSD_USER" ] || TAOSD_USER=root
TAOSD_GROUP="$(id -gn "$TAOSD_USER")"

ADAPTER_USER="$(systemctl show taosadapter -p User --value)"
[ -n "$ADAPTER_USER" ] || ADAPTER_USER=root
ADAPTER_GROUP="$(id -gn "$ADAPTER_USER")"

KEEPER_USER="$(systemctl show taoskeeper -p User --value)"
[ -n "$KEEPER_USER" ] || KEEPER_USER=root
KEEPER_GROUP="$(id -gn "$KEEPER_USER")"

install -d -m 0750 -o "$TAOSD_USER" -g "$TAOSD_GROUP" \
  /data/tdengine/data /data/tdengine/log/taosd /data/tdengine/tmp
install -d -m 0750 -o "$ADAPTER_USER" -g "$ADAPTER_GROUP" \
  /data/tdengine/log/adapter
install -d -m 0750 -o "$KEEPER_USER" -g "$KEEPER_GROUP" \
  /data/tdengine/log/keeper
install -d -m 0750 -o root -g root /backup/tdengine

for link in \
  /data/module/tdengine \
  /data/module/tdengine-3.4.1.13/taos/data \
  /data/module/tdengine-3.4.1.13/taos/log; do
  if [ -e "$link" ] && [ ! -L "$link" ]; then
    echo "ERROR: expected symlink but found real path: $link" >&2
    exit 1
  fi
done

ln -sfn /data/module/tdengine-3.4.1.13/taos /data/module/tdengine

# installer 初始建立的 data/log 链接仍指向 /var，配置迁移后同步修正，避免误导巡检
ln -sfn /data/tdengine/data /data/module/tdengine-3.4.1.13/taos/data
ln -sfn /data/tdengine/log/taosd /data/module/tdengine-3.4.1.13/taos/log

namei -om /data/tdengine/data
readlink -f /data/module/tdengine
readlink -f /data/module/tdengine/data
readlink -f /data/module/tdengine/log
```

禁止对 `/data/module/tdengine*`、`/etc/taos` 或 `/data` 递归执行无差别 `chmod -R 777`、`chown -R`。安装脚本会把初始 `/var/log/taos` 设置为 `777`；配置迁移后应确认没有服务继续使用该目录，并将其收紧为 `0750`，不得照搬此宽权限到生产目录。

---

## 7. 制品准备与校验

### 7.1 Generic 主制品登记

```bash
PKG_FILE=tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz
OFFICIAL_MD5=4c0fc6e31d59107a5125d5284fc58c32
OFFICIAL_SIZE=300274656
AUDITED_SHA256=1b8cba22f1ea510ec72863041413ad86970ece2501d15c6eff482db189add515
```

官方下载地址：

```text
https://downloads.taosdata.com/tdengine-tsdb-oss/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz
```

生产服务器不直接访问公网下载。推荐由受控运维终端下载，完成恶意文件扫描、MD5/SHA256 和脚本审计后上传企业制品库，再从内网分发到服务器。

### 7.2 完整性和企业哈希校验

```bash
cd /data/software/tdengine/3.4.1.13
test -f "$PKG_FILE"

# 字节数仅作辅助核对
test "$(stat -c '%s' "$PKG_FILE")" -eq "$OFFICIAL_SIZE"

# 与官网下载中心 MD5 交叉核验
echo "${OFFICIAL_MD5}  ${PKG_FILE}" | md5sum -c -

# 企业制品库必须维护独立、不可变的 SHA256 清单
echo "${AUDITED_SHA256}  ${PKG_FILE}" | sha256sum -c -
sha256sum -c SHA256SUMS
```

`AUDITED_SHA256` 是本文编制时对官方下载文件的实算结果，不是官网发布的签名或哈希。企业制品入库时仍应独立计算并生成自己的 `SHA256SUMS`。MD5 只能发现常见传输损坏，不能单独证明制品来源可信；生产批准依据应同时包括 HTTPS 官方来源、企业 SHA256 清单、制品库审计和安装脚本审计记录。

### 7.3 解压前安全检查

```bash
# 确认归档可读
gzip -t "$PKG_FILE"

# 查看顶层结构
tar -tzf "$PKG_FILE" | head -100

# 禁止出现绝对路径或父目录穿越
if tar -tzf "$PKG_FILE" | grep -Eq '(^/|(^|/)\.\.(/|$))'; then
  echo 'ERROR: archive contains unsafe paths' >&2
  exit 1
fi
```

官方包的预期顶层目录为 `tdengine-tsdb-oss-3.4.1.13/`，包含 `install.sh`、`start-all.sh`、`stop-all.sh`、`package.tar.gz`、`driver/`、`connector/` 和 `examples/`。

### 7.4 解压并审计安装脚本

```bash
umask 027
tar -zxf "$PKG_FILE"
cd tdengine-tsdb-oss-3.4.1.13

sha256sum install.sh start-all.sh stop-all.sh package.tar.gz
./install.sh -h

# 归档安装脚本涉及的系统路径和高风险操作
grep -nE '/usr/local|/usr/bin|/usr/lib|/etc/|/var/lib|/var/log|systemd|systemctl|chmod 777|rm -rf|ln -s' install.sh \
  > install-script-audit.txt

# 查看内层包的服务、配置和卸载脚本
tar -tzf package.tar.gz \
  | grep -E '^(cfg/|bin/remove\.sh$|init\.d/)' \
  > package-system-files.txt
```

3.4.1.13 实包审计确认：

- `install.sh` 共 1647 行，支持 `-d` 自定义安装父目录；
- `-d /data/module/tdengine-3.4.1.13` 的实际程序目录为 `/data/module/tdengine-3.4.1.13/taos`；
- root 安装仍会修改 `/usr/bin`、`/usr/lib*`、`/usr/include`、`/etc/taos` 和 `/etc/systemd/system`；
- 初始数据和日志目录仍为 `/var/lib/taos`、`/var/log/taos`；
- 安装脚本会启用四个 systemd unit，但不会在本文配置完成前主动启动它们；
- unit 未设置 `User=`，因此服务默认以 root 运行；
- 脚本明确拒绝在 CentOS/RHEL 7 systemd 219 上执行非 root 安装；
- 已有配置不会直接覆盖，新模板写为 `.new`；
- 卸载脚本默认保留配置、数据和日志，只有显式确认或 `-e yes` 才删除。

将 `install-script-audit.txt`、`package-system-files.txt` 和四个 SHA256 一并归档到部署工单。

### 7.5 RPM 备选制品校验

若变更评审最终选择 RPM，仍必须执行以下校验：

```bash
RPM_FILE=tdengine-tsdb-oss-3.4.1.13-linux-x64.rpm
RPM_OFFICIAL_MD5=e1ad92cbdcc3d459243e3311e7b259e4

echo "${RPM_OFFICIAL_MD5}  ${RPM_FILE}" | md5sum -c -
sha256sum -c SHA256SUMS
rpm -Kv "$RPM_FILE"
rpm -qp --qf 'NAME=%{NAME}\nVERSION=%{VERSION}\nRELEASE=%{RELEASE}\nARCH=%{ARCH}\n' "$RPM_FILE"
rpm -qpR "$RPM_FILE"
rpm -qlp "$RPM_FILE" | less
```

RPM 未提供可验证签名时，应记录“来源依赖 HTTPS 下载链和企业制品库控制”。禁止使用 `--nodeps` 强行安装。

### 7.6 安装前系统快照

```bash
mkdir -p /var/tmp/tdengine-precheck
rpm -qa | sort > /var/tmp/tdengine-precheck/rpm-before.txt
systemctl list-unit-files > /var/tmp/tdengine-precheck/systemd-before.txt
ss -lntup > /var/tmp/tdengine-precheck/ports-before.txt
find /usr/bin /usr/lib /usr/lib64 /usr/include /etc/systemd/system \
  -maxdepth 1 -iname '*taos*' -ls \
  > /var/tmp/tdengine-precheck/taos-files-before.txt 2>/dev/null || true
```

---

## 8. 安装 TDengine

### 8.1 安装前确认

```bash
cd /data/software/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13

test "$(id -u)" -eq 0 || echo '后续安装命令必须使用 sudo/root'
test "$(hostname)" = 'prod-tsdb-01'
getent hosts prod-tsdb-01

command -v taos taosd taosadapter taoskeeper 2>/dev/null || true
rpm -qa | grep -Ei 'tdengine|taos' || true
```

如果已经存在 TDengine 命令、RPM 或数据，必须先判断是升级还是历史残留。`install.sh` 检测到现有 `taosd` 后会进入更新逻辑，不能当成全新安装继续执行。

### 8.2 使用自定义版本目录安装

```bash
install -d -m 0755 -o root -g root \
  /data/module \
  /data/module/tdengine-3.4.1.13

cd /data/software/tdengine/3.4.1.13/tdengine-tsdb-oss-3.4.1.13

sudo ./install.sh \
  -v server \
  -e no \
  -d /data/module/tdengine-3.4.1.13
```

参数解释：

- `-v server`：安装服务端和客户端组件；
- `-e no`：关闭交互式 FQDN/集群询问，使用已固化的系统 hostname，便于审计和自动化；
- `-d`：指定安装父目录，脚本会自动追加 `/taos`。

不要把 `-d` 写成最终的 `/data/module/tdengine-3.4.1.13/taos`，否则实际目录会多嵌套一层 `/taos/taos`。

### 8.3 安装结果核验

```bash
test -x /data/module/tdengine-3.4.1.13/taos/bin/taosd
test -f /data/module/tdengine-3.4.1.13/taos/.install_path
cat /data/module/tdengine-3.4.1.13/taos/.install_path

command -v taos taosd taosdump taosadapter taoskeeper
readlink -f "$(command -v taosd)"
taos -V
taosd -V
taosdump -V

systemctl list-unit-files | grep -E '^taos'
systemctl is-enabled taosd taosadapter taoskeeper taos-explorer
systemctl is-active taosd taosadapter taoskeeper taos-explorer || true

systemctl cat taosd | grep -E '^(ExecStart|ExecStartPre|User|Group)='
systemctl cat taosadapter | grep -E '^(ExecStart|User|Group)='
systemctl cat taoskeeper | grep -E '^(ExecStart|User|Group)='

ls -ld \
  /data/module/tdengine-3.4.1.13/taos \
  /etc/taos /var/lib/taos /var/log/taos
```

预期 systemd 的 `ExecStart` 指向 `/data/module/tdengine-3.4.1.13/taos/bin/` 下的对应程序，并通过 `-c` 指向 `/etc/taos` 中的配置。此时服务应为 enabled，但在本文显式启动前应保持 inactive。

### 8.4 收敛安装脚本产生的路径和权限

执行第 6.3 节创建生产目录和当前版本软链接，然后确认：

```bash
chmod 0750 /var/log/taos

readlink -f /data/module/tdengine
readlink -f /data/module/tdengine/data
readlink -f /data/module/tdengine/log

stat -c '%U:%G %a %n' \
  /data/module/tdengine-3.4.1.13/taos \
  /etc/taos /var/lib/taos /var/log/taos \
  /data/tdengine/data /data/tdengine/log/taosd
```

`/var/lib/taos` 和 `/var/log/taos` 是安装脚本的初始目录。完成第 10 节配置后，生产数据和主日志应写入 `/data/tdengine`；保留的空目录用于兼容检查，不得被误认为当前生产数据目录。

### 8.5 首次启动保护

```bash
systemctl stop taos-explorer taoskeeper taosadapter taosd 2>/dev/null || true
systemctl disable taos-explorer 2>/dev/null || true

# 首次初始化前必须为空
find /data/tdengine/data -mindepth 1 -maxdepth 1 -print

# 安装后配置模板有变化时会生成 .new，必须人工评审
find /etc/taos -maxdepth 1 -type f -name '*.new' -ls
```

数据目录有输出时必须确认来源。不得把旧版本数据目录直接交给 3.4.1.13 初始化，也不得用 `.new` 文件直接覆盖生产配置。

---

## 9. 端口与网络访问控制

### 9.1 端口清单

| 端口 | 协议 | 组件/用途 | 单体策略 |
|---:|---|---|---|
| `6030` | TCP | taosd Native/节点端点 | Native 客户端需要时按来源开放 |
| `6041` | TCP | taosAdapter REST/WebSocket | **应用推荐入口**，按来源开放 |
| `6043` | TCP | taosKeeper 健康检查/Prometheus | 仅本机和监控网开放 |
| `6044` | TCP/UDP | StatsD | 默认不开放 |
| `6045` | TCP/UDP | collectd | 默认不开放 |
| `6046` | TCP | OpenTSDB Telnet | 默认不开放 |
| `6047` | TCP | collectd/OpenTSDB | 默认不开放 |
| `6048` | TCP | Icinga2/OpenTSDB | 默认不开放 |
| `6049` | TCP | tcollector/OpenTSDB | 默认不开放 |
| `6060` | TCP | taosExplorer | 默认停用，不开放 |

端口方案至少有以下三种：

| 方案 | 说明 | 评价 |
|---|---|---|
| 方案 A：保留官方端口 | 客户端和组件配置最少，靠白名单隔离 | **推荐** |
| 方案 B：整体改用自定义端口 | 降低与既有服务冲突 | 多组件、SDK、监控和文档必须同步修改 |
| 方案 C：服务保留默认端口，入口 NAT/代理映射 | 内部兼容，外部可统一规划 | 排障链路复杂，Native/UDP 不宜随意代理 |

**推荐方案 A。** 修改默认端口不是有效安全控制；TDengine 涉及多个互相关联端口，保留官方端口并执行精确来源白名单，兼容性和可维护性更好。该例外已登记到《服务器端口统一规划规范.md》。

### 9.2 端口冲突检查

```bash
for p in 6030 6041 6043 6044 6045 6046 6047 6048 6049 6060; do
  ss -lntup | grep -E ":${p}\b" || true
done
```

### 9.3 网络边界方案

| 方案 | 适用环境 | 结论 |
|---|---|---|
| 方案 A：云安全组/硬件防火墙/统一 ACL | 已有集中网络边界 | **与服务器初始化基线一致，推荐** |
| 方案 B：主机 firewalld rich rule | 安全规范要求主机防火墙 | 可用，必须按来源 IP 放行 |
| 方案 C：iptables 白名单 | firewalld 关闭但必须主机过滤 | 需统一持久化和审计，不建议临时手工散落规则 |

仓库初始化手册默认关闭 firewalld，并由外层边界承载访问控制。不要为了本组件单独改变整机基线；若现场明确启用 firewalld，可按以下模板执行。

### 9.4 firewalld 放行模板

```bash
systemctl is-active firewalld

# 应用仅访问 taosAdapter；每个来源网段单独登记
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<APP_CIDR>" port protocol="tcp" port="6041" accept'

# Native 客户端确有需要时才开放 6030
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<NATIVE_CLIENT_CIDR>" port protocol="tcp" port="6030" accept'

# Prometheus 仅访问 taosKeeper
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<MONITOR_CIDR>" port protocol="tcp" port="6043" accept'

firewall-cmd --reload
firewall-cmd --list-rich-rules
```

禁止执行：

```bash
# 禁止：对所有来源开放整个端口段
# firewall-cmd --permanent --add-port=6030-6060/tcp
```

### 9.5 访问矩阵

| 来源 | 目标端口 | 是否允许 |
|---|---:|---|
| 业务应用网段 | `6041/TCP` | 是 |
| 使用 Native 驱动的应用 | `6030/TCP` | 按需 |
| Prometheus | `6043/TCP` | 是 |
| 堡垒机/运维网 | `6030`、`6041`、`6043` | 按需 |
| 公网 | 全部 | 否 |
| 普通办公网 | `6060` | 否；确需 Explorer 时走 VPN/堡垒机 |

---

## 10. 生产配置

### 10.1 配置变更原则

1. 先备份 Generic 安装脚本生成的完整配置，不覆盖未知的新版本参数或 `.new` 文件；
2. 只修改经过 3.4.1 官方文档核验的参数；
3. 所有 FQDN、端口、路径和密码必须替换现场值；
4. 配置文件不得进入包含真实密码的 Git 仓库；
5. 修改后先做前台/日志检查，再设置开机自启。

```bash
cp -a /etc/taos/taos.cfg "/etc/taos/taos.cfg.$(date +%F_%H%M%S).bak"
cp -a /etc/taos/taosadapter.toml "/etc/taos/taosadapter.toml.$(date +%F_%H%M%S).bak"
cp -a /etc/taos/taoskeeper.toml "/etc/taos/taoskeeper.toml.$(date +%F_%H%M%S).bak"
```

### 10.2 taosd 配置

编辑 `/etc/taos/taos.cfg`，保留原文件注释并设置以下项：

> 实包 `install.sh` 会追加 `monitor 1` 和大小写写作 `monitorFQDN` 的配置。编辑时必须把它们归并为本节所示的唯一有效项，避免与后续追加内容重复。

```ini
# ===== identity and network =====
firstEp             prod-tsdb-01:6030
fqdn                prod-tsdb-01
serverPort          6030

# ===== storage =====
dataDir             /data/tdengine/data
logDir              /data/tdengine/log/taosd
tempDir             /data/tdengine/tmp

# 最低保留空间是保护线，不是容量告警替代品；按磁盘规模调整
minimalDataDirGB    20
minimalLogDirGB     5
minimalTmpDirGB     5

# ===== log =====
asyncLog            1
numOfLogLines       10000000

# ===== monitoring =====
monitor             1
monitorFqdn         prod-tsdb-01
monitorPort         6043
monitorInterval     30
```

说明：

- `dataDir`、`logDir`、`tempDir` 和端口参数需要重启；
- `supportVnodes` 默认值为 CPU 核数的两倍加 5，本文不盲目覆盖；
- `timezone`、`locale`、`charset` 默认取系统值，统一由 OS 基线控制；
- `firstEp` 对首次初始化非常关键，单体也固定为本机稳定 FQDN；
- 初始化后修改 `fqdn`、`serverPort` 或主机名属于高风险变更。

配置项去重检查：

```bash
grep -niE '^(firstEp|fqdn|serverPort|dataDir|logDir|tempDir|minimalDataDirGB|minimalLogDirGB|minimalTmpDirGB|asyncLog|numOfLogLines|monitor|monitorFqdn|monitorPort|monitorInterval)[[:space:]]' /etc/taos/taos.cfg
```

每个参数应只有一个有效值；发现重复时人工合并，不要依赖“最后一行覆盖”的偶然行为。

### 10.3 taosAdapter 配置

编辑 `/etc/taos/taosadapter.toml`，至少确认：

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

连接池默认 `pool.maxConnect` 为 CPU 核数的两倍，官方建议保持默认。未经压测不要扩大连接池；应用侧也必须限制连接数。

逐项检查并关闭未使用的数据入口：

```toml
[collectd]
enable = false

[statsd]
enable = false

[opentsdb_telnet]
enable = false
```

不同 3.4 维护版本的 TOML 模板可能存在差异。只修改当前 Generic 包实际生成且官方 `taosadapter --help`/3.4.1 文档支持的段落；不存在的段落不要机械追加。

### 10.4 taosKeeper 配置

编辑 `/etc/taos/taoskeeper.toml`：

```toml
instanceId = 64
host = "127.0.0.1"
port = 6043
RotationInterval = "15s"

[tdengine]
host = "127.0.0.1"
port = 6041
username = "root"
password = "<修改后的生产密码>"
usessl = false

[metrics]
prefix = "taos"
tables = []

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

安全处理：

```bash
KEEPER_USER="$(systemctl show taoskeeper -p User --value)"
[ -n "$KEEPER_USER" ] || KEEPER_USER=root
KEEPER_GROUP="$(id -gn "$KEEPER_USER")"
chown root:"$KEEPER_GROUP" /etc/taos/taoskeeper.toml
chmod 0640 /etc/taos/taoskeeper.toml
stat -c '%U:%G %a %n' /etc/taos/taoskeeper.toml
```

taosKeeper 官方示例需要 TDengine 用户名和密码。OSS 权限模型与 Enterprise 不同；若版本和实测允许使用满足监控查询权限的专用账号，应优先使用专用账号，否则需把 root 明文凭据风险纳入登记并严格限制文件读取。

### 10.5 Explorer 策略

| 方案 | 说明 | 结论 |
|---|---|---|
| 方案 A：不启动 Explorer | 减少 Web 管理面和 6060 暴露 | **推荐** |
| 方案 B：仅本机监听，经 SSH 隧道访问 | 便于临时运维 | 使用后关闭 |
| 方案 C：内网长期开放 | 使用方便 | 风险最高，需 VPN、认证、反向代理和审计 |

```bash
systemctl disable --now taos-explorer 2>/dev/null || true
```

---

## 11. 启动、初始化与改密

### 11.1 启动顺序

官方快速开始提供 `start-all.sh`，但 3.4.1.13 实包脚本会依次启动 `taosd`、`taosadapter`、`taos-explorer` 和 `taoskeeper`。生产默认停用 Explorer，因此禁止直接使用 `start-all.sh`，应显式控制组件：

```bash
systemctl daemon-reload

systemctl enable --now taosd
systemctl status taosd --no-pager -l

systemctl enable --now taosadapter
systemctl status taosadapter --no-pager -l

# 首次改密并同步 keeper 配置后再启动 keeper
systemctl disable taos-explorer 2>/dev/null || true
```

检查日志和监听：

```bash
journalctl -u taosd -u taosadapter --since '-10 min' --no-pager
ss -lntp | grep -E ':(6030|6041)\b'
tail -n 100 /data/tdengine/log/taosd/* 2>/dev/null
tail -n 100 /data/tdengine/log/adapter/* 2>/dev/null
```

### 11.2 首次登录与立即改密

官方默认账号为 `root`，默认密码为 `taosdata`。首次启动后必须立即在受控终端修改。

```bash
taos -h 127.0.0.1 -P 6030 -u root -p
```

输入默认密码后执行：

```sql
ALTER USER root PASS '<至少16位随机强密码>';
QUIT;
```

3.4.1 官方文档要求密码长度 8～255，且至少包含大写、小写、数字、特殊字符四类中的三类。生产标准提高为至少 16 位随机密码，禁止关闭强密码校验。

重新登录验证：

```bash
taos -h 127.0.0.1 -P 6030 -u root -p
```

密码不得出现在命令行参数、Shell 历史、Git、工单正文或聊天记录中。应存入企业密码库。

### 11.3 同步 Keeper 凭据并启动

修改 `/etc/taos/taoskeeper.toml` 的 `[tdengine].password`，再执行：

```bash
chmod 0640 /etc/taos/taoskeeper.toml
systemctl enable --now taoskeeper
systemctl status taoskeeper --no-pager -l
curl -fsS http://127.0.0.1:6043/check_health
```

### 11.4 基础数据库示例

生产数据库参数必须来自容量评审。以下只展示语法，不可原样上线：

```sql
CREATE DATABASE prod_iot
  KEEP 365d
  REPLICA 1
  VGROUPS <压测确定值>;
```

决策要求：

- 单体只能设置 `REPLICA 1`；
- `KEEP` 是数据生命周期策略，修改前需要业务确认；
- `VGROUPS` 按 CPU、内存和压测结果确定；
- 不要使用测试库名、默认保留期或未评审参数承载生产数据。

---

## 12. 备份与恢复

### 12.1 OSS 备份能力边界

官方确认 `taosdump`：

- 可连接正在运行的 TDengine；
- 可备份全部数据库、指定数据库、表或时间范围；
- 使用 Apache Avro 文件；
- 导出不支持断点续传，中断后需删除本次不完整文件并重做；
- 导入支持中断后重试，但可能提示表已存在；
- 已有文件的输出目录不会被覆盖，同一目录只用于一次备份；
- OSS 不提供 Enterprise 持续增量备份和时间点恢复。

### 12.2 备份方案选择

| 方案 | RPO 特征 | 优点 | 风险 | 结论 |
|---|---|---|---|---|
| 方案 A：每日逻辑全量 | 最坏约 24 小时 | 简单、官方支持、易验证 | 大库窗口长 | **中小单体推荐** |
| 方案 B：按 `-S/-E` 时间分段 | 可缩短每段窗口 | 适合大库；失败只重做分段 | 目录和恢复顺序管理复杂 | 大数据量推荐 |
| 方案 C：停机冷快照/文件系统快照 | 取决于快照周期 | 快速保留整盘状态 | 跨版本、跨主机恢复兼容风险高 | 仅作第二保��层，不替代 taosdump |

推荐采用 **A 或 B + 异机复制 + 定期恢复演练**。只把 `/data/tdengine/data` 做在线 `cp -a` 不属于本文认可的生产备份。

### 12.3 全量备份

```bash
BACKUP_ROOT=/backup/tdengine
BACKUP_ID="$(date +%F_%H%M%S)"
BACKUP_DIR="${BACKUP_ROOT}/${BACKUP_ID}"

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

备份后生成外置清单：

```bash
find "$BACKUP_DIR" -type f -print0 \
  | sort -z \
  | xargs -0 sha256sum > "${BACKUP_DIR}.sha256"

du -sh "$BACKUP_DIR"
grep -RniE 'error|failed' "$BACKUP_DIR" 2>/dev/null || true
```

将备份目录和 `.sha256` 清单复制到独立备份存储，并在复制后再次校验。备份任务不得在命令行传入明文密码。

### 12.4 分段备份

```bash
taosdump \
  -h 127.0.0.1 -P 6030 -u root -p \
  -D prod_iot \
  -S '2026-07-12 00:00:00.000+0800' \
  -E '2026-07-12 23:59:59.999+0800' \
  -d snappy -T 8 \
  -o /backup/tdengine/prod_iot_2026-07-12
```

分段边界必须统一时区，避免重叠或缺口。元数据/Schema 应另做定期全量备份，不能只保留数据时间片。

### 12.5 恢复前检查

恢复是写操作，必须先明确：

- 恢复到原库还是隔离库；
- 目标库是否已存在、是否允许覆盖或合并；
- 备份版本、服务端版本和连接器版本；
- 可用磁盘空间；
- 业务停写窗口；
- 回退方案。

校验备份：

```bash
sha256sum -c /backup/tdengine/<BACKUP_ID>.sha256
taosdump -I -i /backup/tdengine/<BACKUP_ID> | head -100
```

### 12.6 隔离库恢复演练

优先恢复到独立测试实例；无法提供独立实例时，使用 `-W` 改名到隔离数据库：

```bash
taosdump \
  -i /backup/tdengine/<BACKUP_ID> \
  -h 127.0.0.1 \
  -P 6030 \
  -u root \
  -p \
  -W 'prod_iot=restore_drill_20260713'
```

恢复后验证：

```sql
SHOW DATABASES;
USE restore_drill_20260713;
SHOW STABLES;
SHOW TABLES;
SELECT COUNT(*) FROM <代表性超级表>;
SELECT MIN(ts), MAX(ts) FROM <代表性超级表>;
```

数量、时间边界、抽样值、Schema 和业务查询均通过后，恢复演练才算成功。

### 12.7 恢复异常处理

- 出现 `WAL size exceeds limit` 时，按官方建议减小 `-B`；
- 导出失败后删除本次新建的不完整备份目录，再重新执行；
- 导入重试出现“table already exists”需结合日志判断，不能一律忽略其他错误；
- 不要在原生产库上反复试恢复；
- 恢复报告必须记录备份 ID、开始/结束时间、版本、数据量、错误和 RTO。

---

## 13. 监控与告警

### 13.1 监控方案选择

| 方案 | 说明 | 结论 |
|---|---|---|
| 方案 A：taosKeeper `/metrics/v2` + Prometheus/Grafana | 官方推荐端点，指标完整、内存缓存 | **推荐** |
| 方案 B：taosKeeper 监控库 SQL 巡检 | 直接查看 `log` 数据库 | 作为排障补充 |
| 方案 C：仅 OS 进程/端口监控 | 部署简单 | 无法覆盖 vnode、请求和 TDengine 内部状态，不可单独使用 |

### 13.2 健康检查

```bash
systemctl is-active taosd taosadapter taoskeeper
curl -fsS http://127.0.0.1:6043/check_health
curl -fsS 'http://127.0.0.1:6043/metrics/v2?max_age=30' | head
curl -fsS http://127.0.0.1:6041/-/ping
```

若 `/-/ping` 在实际 3.4.1.13 包中不可用，以 WebSocket/REST 的受控 SQL 探针和 systemd 状态替代，并记录实际健康检查接口。

### 13.3 Prometheus 配置

```yaml
scrape_configs:
  - job_name: "tdengine-single"
    metrics_path: "/metrics/v2"
    params:
      max_age: ["30"]
    static_configs:
      - targets: ["prod-tsdb-01:6043"]
```

单体只有一个 Keeper 目标。Prometheus 到 `6043/TCP` 只允许监控网段访问。

### 13.4 重点指标和告警

| 类别 | 指标/检查 | 建议告警 |
|---|---|---|
| 存活 | `taosd_dnodes_status_status` | 非 `ready=1` 立即严重告警 |
| CPU | `taosd_dnodes_info_cpu_engine`、系统 CPU | 持续高于 50% 预警，结合压测扩容 |
| 内存 | Engine 内存、系统可用内存、OOM | 可用内存持续低或 OOM 立即告警 |
| 磁盘 | data/log/tmp 使用率和剩余 GB | 70% 警告，80% 严重 |
| I/O | 读写速率、await、util | 与基线偏离持续 15 分钟告警 |
| vnode/vgroup | 总数、存活数、状态 | 非 ready 或数量异常告警 |
| 连接 | 连接数、Adapter 连接池等待 | 接近压测上限或持续等待告警 |
| 错误 | taosd 错误数、错误日志增长 | 持续增长告警 |
| 备份 | 最近成功时间、大小、校验、异机复制 | 超过 RPO 未成功则严重告警 |
| 时间 | chrony 偏差 | 超过企业时间基线告警 |

官方建议常态 CPU 保持在 50% 以下，因此本文不把 80% 当成首个容量告警点。

### 13.5 SQL 巡检

```sql
SHOW DNODES;
SHOW DATABASES;
SELECT * FROM information_schema.ins_dnodes;
SELECT * FROM information_schema.ins_vnodes;
USE log;
SHOW STABLES;
```

`log` 监控数据库默认保留 90 天，容量和保留期需纳入巡检。

---

## 14. 生产验证清单

### 14.1 安装与资产

- [ ] OS 为 CentOS 7.9 x86_64；
- [ ] Generic 文件名为 `tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz`；
- [ ] 文件大小、官网 MD5 和企业 SHA256 均校验通过；
- [ ] `install.sh`、内层系统文件清单和脚本审计结果已归档；
- [ ] 程序实际安装在 `/data/module/tdengine-3.4.1.13/taos`；
- [ ] `/usr/bin`、`/usr/lib*`、`/usr/include` 和 systemd unit 的 TDengine 链接/文件已登记；
- [ ] `taosd -V`、`taos -V`、`taosdump -V` 已记录；
- [ ] CMDB 已记录主机、版本、FQDN、路径、端口和责任人。

### 14.2 OS 与目录

- [ ] FQDN 唯一、正反向解析满足现场规范；
- [ ] chrony 正常，时区正确；
- [ ] 数据盘独立挂载且容量满足评审；
- [ ] data/log/tmp/backup 目录权限符合实际服务账号；
- [ ] 未使用 `777` 或无差别递归授权；
- [ ] 文件句柄、Swap、THP 按初始化手册验收。

### 14.3 配置

- [ ] `firstEp`、`fqdn`、`serverPort` 正确；
- [ ] data/log/tmp 指向规划目录；
- [ ] 最低磁盘保留参数合理；
- [ ] `monitor=1` 且 Keeper 地址正确；
- [ ] Adapter `debug=false`；
- [ ] 未使用的 6044～6049 输入插件已关闭；
- [ ] Explorer 默认关闭；
- [ ] Keeper 配置文件由 root 管理、实际服务组可读、权限为 `0640`，且未纳入 Git。

### 14.4 服务与端口

```bash
systemctl status taosd taosadapter taoskeeper --no-pager
systemctl is-enabled taosd taosadapter taoskeeper
ss -lntup | grep -E ':(6030|6041|6043|6044|6045|6046|6047|6048|6049|6060)\b'
```

- [ ] 仅监听已审批端口；
- [ ] 6030/6041 不对公网开放；
- [ ] 6043 仅监控来源可访问；
- [ ] 6060 未监听；
- [ ] 重启服务器后服务自动恢复。

### 14.5 安全

- [ ] root 默认密码已修改；
- [ ] 新密码满足企业强度并存入密码库；
- [ ] 旧密码登录失败；
- [ ] 命令历史、日志和工单中没有明文密码；
- [ ] Keeper 凭据风险已登记；
- [ ] 防火墙/安全组使用精确来源白名单。

### 14.6 读写验证

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

- [ ] Native CLI 读写通过；
- [ ] 业务实际使用的 WebSocket/REST 连接通过；
- [ ] 应用连接池断开重连通过；
- [ ] 中文、时区、毫秒/微秒精度符合数据模型；
- [ ] 典型聚合和时间窗口查询结果正确。

### 14.7 备份恢复与监控

- [ ] 完成一次全量或分段备份；
- [ ] SHA256 清单校验通过；
- [ ] 备份已复制到异机/独立存储；
- [ ] 完成隔离实例或隔离库恢复；
- [ ] 恢复数据量和时间边界正确；
- [ ] `/check_health` 返回 200；
- [ ] Prometheus 成功抓取 `/metrics/v2`；
- [ ] 存活、CPU、磁盘、错误和备份告警已测试。

### 14.8 重启验证

```bash
systemctl restart taosd
systemctl restart taosadapter
systemctl restart taoskeeper
systemctl is-active taosd taosadapter taoskeeper
```

维护窗口内还应执行一次操作系统重启验收，确认挂载、FQDN、服务、端口、数据和监控均恢复。

---

## 15. 日常运维

### 15.1 服务命令

```bash
systemctl status taosd taosadapter taoskeeper --no-pager
systemctl restart taosd
systemctl restart taosadapter
systemctl restart taoskeeper
journalctl -u taosd --since '-30 min' --no-pager
journalctl -u taosadapter --since '-30 min' --no-pager
journalctl -u taoskeeper --since '-30 min' --no-pager
```

停止顺序：先停止写入方，再停止 Keeper、Adapter、taosd。启动顺序相反：taosd、Adapter、Keeper，最后恢复写入方。

### 15.2 资源巡检

```bash
df -hT /data /backup 2>/dev/null
du -sh /data/tdengine/data /data/tdengine/log/* /data/tdengine/tmp
pidstat -p "$(pidof taosd)" 1 5
iostat -x 1 5
ss -s
chronyc tracking
```

### 15.3 配置变更流程

1. 创建工单，说明参数、原因、风险和回滚；
2. 备份三个配置文件并记录哈希；
3. 在预生产验证；
4. 确认参数是动态生效还是需要重启；
5. 维护窗口修改并观察日志；
6. 执行第 14 节相关验证；
7. 失败时恢复备份配置并重启对应服务。

```bash
sha256sum /etc/taos/taos.cfg /etc/taos/taosadapter.toml /etc/taos/taoskeeper.toml
```

---

## 16. 升级与回滚

### 16.1 官方版本规则

TDengine 四段版本号含义为：

```text
[Major+].[Major].[Feature].[Maintenance]
```

官方 3.4.1 文档说明：

- 第四段 Maintenance：仅缺陷修复，支持滚动升级并可回退；
- 第三段 Feature：不支持滚动升级，同一 Major 内可回退，客户端和服务端应同步升级；
- 第二段 Major：不支持滚动升级，升级后不可逆；
- 单体无论版本类型都没有滚动升级，升级期间必然中断服务。

`3.4.1.13 -> 3.4.1.x` 属于 Maintenance 范围，但仍必须阅读目标版本发布说明并做备份恢复演练。

### 16.2 升级前门禁

- 目标版本官方制品和校验信息完整；
- 源版本到目标版本路径得到官方文档或供应方确认；
- 应用驱动兼容性已验证；
- 全量逻辑备份和配置备份已完成并异机保存；
- 隔离恢复演练通过；
- 旧、新 Generic 原始包、企业 SHA256 清单和安装脚本审计记录仍可获取；
- 旧版本程序目录在回滚观察期内保持不变；
- 磁盘至少保留一个完整恢复副本所需空间；
- 明确停机窗口和业务回退判断点。

### 16.3 升级步骤

```bash
# 1. 记录基线
taosd -V
readlink -f "$(command -v taosd)"
systemctl cat taosd taosadapter taoskeeper > /var/tmp/tdengine-units-before-upgrade.txt
sha256sum /etc/taos/taos.cfg /etc/taos/taosadapter.toml /etc/taos/taoskeeper.toml

# 2. 先停止业务写入，再停止组件
systemctl stop taoskeeper
systemctl stop taosadapter
systemctl stop taosd

# 3. 再次确认没有进程和端口
ps -ef | grep -E '[t]aosd|[t]aosadapter|[t]aoskeeper' || true
ss -lntup | grep -E ':(6030|6041|6043)\b' || true

# 4. 解压已经校验和审计的新 Generic 包
TARGET_VERSION=<目标版本>
cd "/data/software/tdengine/${TARGET_VERSION}"
tar -zxf "tdengine-tsdb-oss-${TARGET_VERSION}-linux-x64.tar.gz"
cd "tdengine-tsdb-oss-${TARGET_VERSION}"

# 5. 安装到新版本父目录；脚本最终追加 /taos
install -d -m 0755 -o root -g root "/data/module/tdengine-${TARGET_VERSION}"
sudo ./install.sh -v server -e no -d "/data/module/tdengine-${TARGET_VERSION}"

# installer 会再次启用 Explorer，并可能把初始日志目录放宽为 777，立即恢复生产基线
systemctl disable taos-explorer
chmod 0750 /var/log/taos

# 6. 确认 systemd 和全局命令已经切到新版本
readlink -f "$(command -v taosd)"
systemctl cat taosd | grep '^ExecStart='
systemctl cat taosadapter | grep '^ExecStart='
systemctl cat taoskeeper | grep '^ExecStart='

# 7. 新模板只允许以 .new 形式出现，逐项人工合并
find /etc/taos -maxdepth 1 -type f -name '*.new' -ls

# 8. 更新团队当前版本软链接，并修正 installer 的 data/log 初始链接
for link in \
  /data/module/tdengine \
  "/data/module/tdengine-${TARGET_VERSION}/taos/data" \
  "/data/module/tdengine-${TARGET_VERSION}/taos/log"; do
  if [ -e "$link" ] && [ ! -L "$link" ]; then
    echo "ERROR: expected symlink but found real path: $link" >&2
    exit 1
  fi
done

ln -sfn "/data/module/tdengine-${TARGET_VERSION}/taos" /data/module/tdengine
ln -sfn /data/tdengine/data "/data/module/tdengine-${TARGET_VERSION}/taos/data"
ln -sfn /data/tdengine/log/taosd "/data/module/tdengine-${TARGET_VERSION}/taos/log"

# 9. 启动和验证
systemctl start taosd
systemctl start taosadapter
systemctl start taoskeeper
taosd -V
```

然后完整执行第 14 节的服务、读写、监控和应用验证，再恢复业务写入。

### 16.4 回滚方案

| 方案 | 适用故障 | 风险 | 结论 |
|---|---|---|---|
| 方案 A：恢复旧配置 | 新参数或 `.new` 合并错误 | 不能解决二进制/数据格式问题 | 第一优先 |
| 方案 B：同 Maintenance 重新执行旧 Generic 安装脚本 | 官方确认可逆且旧版本目录/制品完整 | 会重建全局链接和 systemd unit，必须完整验证 | 条件允许时使用 |
| 方案 C：重装旧版本并从逻辑备份恢复到空目录 | 需要确定性恢复 | RTO 最长、丢失备份后新增数据 | **最稳妥的最终回滚** |

方案 B 只有在目标路径经官方确认、预生产验证且备份可用时才能执行。示意流程：

```bash
OLD_VERSION=<旧版本>

systemctl stop taoskeeper taosadapter taosd

cd "/data/software/tdengine/${OLD_VERSION}/tdengine-tsdb-oss-${OLD_VERSION}"
sudo ./install.sh -v server -e no -d "/data/module/tdengine-${OLD_VERSION}"

systemctl disable taos-explorer
chmod 0750 /var/log/taos

for link in \
  /data/module/tdengine \
  "/data/module/tdengine-${OLD_VERSION}/taos/data" \
  "/data/module/tdengine-${OLD_VERSION}/taos/log"; do
  if [ -e "$link" ] && [ ! -L "$link" ]; then
    echo "ERROR: expected symlink but found real path: $link" >&2
    exit 1
  fi
done

ln -sfn "/data/module/tdengine-${OLD_VERSION}/taos" /data/module/tdengine
ln -sfn /data/tdengine/data "/data/module/tdengine-${OLD_VERSION}/taos/data"
ln -sfn /data/tdengine/log/taosd "/data/module/tdengine-${OLD_VERSION}/taos/log"

readlink -f "$(command -v taosd)"
systemctl cat taosd | grep '^ExecStart='

systemctl start taosd
systemctl start taosadapter
systemctl start taoskeeper
```

若升级跨 Feature/Major，或启动后出现数据格式、元数据异常，应停止原地反复启动，采用以下流程：

1. 保留故障现场和日志；
2. 安装旧版本到干净服务器或清空后的新数据目录；
3. 恢复旧配置中仍适用的部分；
4. 从升级前 `taosdump` 备份恢复；
5. 执行完整业务验证；
6. 切回业务流量。

### 16.5 卸载和旧版本目录清理警告

Generic 包安装后提供 `rmtaos` 和版本目录内的 `uninstall.sh`。3.4.1.13 实包语义为：

```text
-e no     卸载程序、全局链接和服务，保留配置、数据和日志
-e yes    同时删除配置、数据和日志
-d PATH   指定最终安装目录，例如 /data/module/tdengine-3.4.1.13/taos
```

完整下线产品且需要保留数据时，才可在备份验证和停机后执行：

```bash
sudo /data/module/tdengine-3.4.1.13/taos/uninstall.sh \
  -e no \
  -d /data/module/tdengine-3.4.1.13/taos
```

禁止在生产环境使用 `-e yes`，除非执行的是经过双人复核的数据销毁工单。

更重要的是，卸载脚本会删除共享的 `/usr/bin`、动态库链接和 systemd unit。存在多个并存版本时，不能用旧版本 `uninstall.sh` 清理旧目录，否则可能破坏当前版本。旧版本清理有三种方案：

| 方案 | 说明 | 结论 |
|---|---|---|
| 方案 A：回滚观察期后只清理已确认未引用的旧版本目录 | 不触碰当前全局链接和 unit | **推荐，需核对绝对路径** |
| 方案 B：运行旧版本卸载脚本后立即重装当前版本 | 系统级副作用较大 | 不推荐，仅测试环境使用 |
| 方案 C：保留旧版本目录 | 占用磁盘但回滚最直接 | 空间充足时可保留更久 |

清理前必须确认：

```bash
OLD_DIR=/data/module/tdengine-<旧版本>/taos
readlink -f /data/module/tdengine
readlink -f "$(command -v taosd)"
systemctl cat taosd | grep '^ExecStart='
find /usr/bin /usr/lib /usr/lib64 /usr/include -maxdepth 1 -type l \
  -lname "${OLD_DIR}/*" -ls
```

只要任一当前链接或 `ExecStart` 仍指向 `OLD_DIR`，就禁止删除。任何情况下都禁止把 `/data/tdengine/data` 纳入程序目录清理命令。

---

## 17. 风险与限制

| 风险 | 影响 | 控制措施 |
|---|---|---|
| 单体单点 | 主机、磁盘、进程故障即中断 | 明确 SLA；异机备份；后续评估集群 |
| CentOS 7 EOL | 无常规安全更新，生态逐步停止支持 | 强网络隔离、漏洞补偿、制定 OS 迁移计划 |
| 3.4.1 非官网标注 LTS | 生命周期和长期修复预期弱于 LTS | 候选门禁；评估 3.3.6 LTS 或后续 3.4.2 |
| 3.4.1.13 发布说明链接失效 | 无法完整审查变更和已知缺陷 | 上线前向官方/社区补齐；保留核验记录 |
| 官网只展示 MD5 | MD5 不能证明抗篡改来源 | 企业 SHA256、HTTPS、制品库不可变审计、安装脚本和内层文件清单审计 |
| Generic 安装脚本是系统级安装器 | 会更新全局命令、动态库链接、配置和 systemd unit；升级时可能重新启用 Explorer、放宽初始日志权限 | 安装前后快照、带版本程序目录、`.new` 人工合并、升级后立即恢复安全基线 |
| CentOS 7 服务默认 root 运行 | 服务漏洞影响面较大 | 严格网络隔离、最小端口、目录权限、停用 Explorer；专用账号改造需专项验证 |
| OSS 备份能力有限 | 无持续增量、PITR；大库全量窗口长 | 分段备份、异机复制、恢复演练、评估 Enterprise/外部方案 |
| Keeper 保存数据库密码 | 配置泄露可暴露高权限凭据 | `0640`、最小读取权限、密码库、优先专用监控账号 |
| OSS 与 Enterprise 文档混排 | 误把 Enterprise 能力写入 OSS 方案 | 本文明确排除 RBAC、审计、白名单、加密、增量备份等能力 |
| Native/REST 暴露 | 暴力认证、未授权访问、明文链路风险 | 不对公网开放；来源白名单；VPN/专网；必要时代理 TLS |
| Explorer 管理面 | 增加 Web 攻击面和运维误操作 | 默认停用；临时隧道访问 |
| FQDN/主机名变更 | 节点身份和连接异常 | 初始化前固化；变更需专项方案 |
| 磁盘打满 | 写入停止、日志丢失或实例异常 | 70/80% 告警、最低保留空间、容量扩展预案 |
| 大 REST 结果集 | Adapter/客户端内存溢出 | WebSocket 流式读取、限制查询时间窗和返回量 |
| 客户端版本不一致 | 协议或行为不兼容 | 驱动矩阵、联调、Feature 升级同步客户端 |

生产评审必须明确接受：单体 TDengine 的可用性上限由单台服务器决定。备份只能改善数据恢复，不能提供服务高可用。

---

## 18. 参考资料

以下资料于 **2026-07-13** 重新核验：

1. [TDengine Download Center](https://tdengine.com/downloads/)：3.4.1.13 OSS RPM、Generic、Docker 文件名、大小、MD5 和安装命令；
2. [TDengine 3.4.1 Supported Platforms](https://docs.tdengine.com/3.4.1/reference/supported/)：CentOS 7.9+、x64 支持矩阵；
3. [TDengine 3.4.1 System Requirements](https://docs.tdengine.com/3.4.1/operation/planning/)：内存公式、CPU/vnode、压缩率、网络和端口；
4. [Deploy from Package](https://docs.tdengine.com/3.4.1/get-started/package/)：安装包部署、systemd 服务和默认账号；
5. [taosd Reference](https://docs.tdengine.com/3.4.1/reference/components/taosd/)：taos.cfg、目录、端口、日志和监控参数；
6. [taosAdapter Reference](https://docs.tdengine.com/3.4.1/reference/components/taosadapter/)：6041、连接池、日志和 Keeper 上报；
7. [taosKeeper Reference](https://docs.tdengine.com/3.4.1/reference/components/taoskeeper/)：配置、健康检查、监控库和 Prometheus `/metrics/v2`；
8. [Data Backup and Restoration](https://docs.tdengine.com/3.4.1/operation/backup/)：OSS taosdump 与 Enterprise 增量备份边界；
9. [taosdump Reference](https://docs.tdengine.com/3.4.1/reference/tools/taosdump/)：备份恢复参数、Avro、分段和断点限制；
10. [Users and Permissions](https://docs.tdengine.com/3.4.1/operation/user/)：默认 root、改密语法和密码规则；
11. [TDengine Release History](https://docs.tdengine.com/3.4.1/releases/)：四段版本规则、升级和回退语义；
12. [Manual Deployment](https://docs.tdengine.com/3.4.1/operation/deployment/manual/)：FQDN、firstEp、serverPort 和启动要求。
13. [TDengine 3.4.1.13 Linux-Generic x64 官方包](https://downloads.taosdata.com/tdengine-tsdb-oss/3.4.1.13/tdengine-tsdb-oss-3.4.1.13-linux-x64.tar.gz)：实包复核 `install.sh`、systemd unit、默认配置、卸载脚本和系统级文件清单。

> 官方网站内容会更新。每次安装或升级都应重新核验目标版本下载页、Release Notes、兼容矩阵和备份恢复说明，不能仅依赖本文历史结论。
