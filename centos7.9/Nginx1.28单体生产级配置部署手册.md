# Nginx 1.31 单体生产级安装配置部署手册

> **Author:** 王有政  
> **Encoding:** UTF-8  
> **日期:** 2026-06-16  
> **适用系统:** CentOS 7.9 (x86_64)  
> **Nginx 版本:** 1.31.1 Mainline  
> **文档定位:** 单体 Nginx 生产级从安装到配置、启动、验证、运维与回滚的一站式部署手册，适用于 Vue 前端静态资源托管、API 反向代理、HTTPS 入口和基础安全加固。

---

## 目录

1. [适用范围与部署原则](#1-适用范围与部署原则)
2. [部署规划](#2-部署规划)
3. [部署前检查](#3-部署前检查)
4. [安装方案选择](#4-安装方案选择)
5. [源码包准备与校验](#5-源码包准备与校验)
6. [编译安装 Nginx](#6-编译安装-nginx)
7. [配置环境变量](#7-配置环境变量)
8. [OS 内核与资源限制配置](#8-os-内核与资源限制配置)
9. [目录、用户与权限](#9-目录用户与权限)
10. [Nginx 标准化配置文件写入](#10-nginx-标准化配置文件写入)
11. [Vue 前端站点配置](#11-vue-前端站点配置)
12. [HTTPS 证书配置](#12-https-证书配置)
13. [systemd 服务配置](#13-systemd-服务配置)
14. [网络边界与访问控制](#14-网络边界与访问控制)
15. [启动与基础验证](#15-启动与基础验证)
16. [生产验证清单](#16-生产验证清单)
17. [运维检查命令](#17-运维检查命令)
18. [前端发布、回滚与缓存刷新](#18-前端发布回滚与缓存刷新)
19. [停止、重启与回滚](#19-停止重启与回滚)
20. [日志切割](#20-日志切割)
21. [常见问题](#21-常见问题)

---

## 1. 适用范围与部署原则

本文用于在 **CentOS 7.9** 上部署 **Nginx 1.31.1 单体生产实例**，适用于以下场景：

- Vue、React、Angular 等前端静态资源生产部署；
- 前后端分离系统的统一 HTTP/HTTPS 入口；
- `/api/` 等路径反向代理到后端服务；
- 单机 Web 网关、内网管理系统入口、轻量级业务系统入口；
- 后续作为负载均衡、HTTPS 终止、灰度发布、静态资源服务的基础模板。

本文不覆盖以下内容：

- Nginx 集群高可用；
- Keepalived 漂移 VIP；
- Kubernetes Ingress；
- OpenResty、Lua 网关开发；
- CDN、WAF、全局负载均衡；
- 大规模多租户网关平台治理。

生产级单体 Nginx 必须遵循以下原则：

1. Nginx worker 进程不得使用 `root` 用户运行；
2. Nginx 必须由 systemd 托管并设置开机自启；
3. 配置文件必须先 `nginx -t` 校验，再 reload 或 restart；
4. 前端静态资源、配置、日志、证书、运行时文件必须目录清晰、权限收敛；
5. Vue history 路由必须配置 `try_files` 回退到 `index.html`；
6. `index.html` 不得强缓存，带 hash 的 JS/CSS/assets 可以长缓存；
7. API 反向代理必须显式传递真实客户端 IP 相关请求头；
8. 生产环境必须启用访问日志和错误日志；
9. 公网系统必须启用 HTTPS，内网系统建议按企业安全要求启用 HTTPS；
10. 不得在生产环境使用 `npm run dev`、`vite preview`、临时 Node 静态服务器托管前端；
11. 不得将证书私钥、业务配置密钥、`.env` 等敏感文件放入 Web 可访问目录；
12. 端口不得直接暴露到公网，必须由外层防火墙、安全组、ACL、VPN、堡垒机或统一网络策略保护；
13. 单体 Nginx 存在单点风险，核心生产系统应结合负载均衡、VIP、CDN、WAF 或多节点架构；
14. Nginx 监听端口默认收敛在非特权端口（HTTP `8088`、HTTPS `8443`），通过外层防火墙做 NAT 映射对外提供服务，不直接监听 `80/443`，目的不是"换端口就安全"，而是减少无差别扫描噪声、避免与系统其他 Web 中间件冲突、为后续 worker 直接绑定预留空间；
15. 公网暴露的 Nginx 必须启用基础限流（`limit_req`、`limit_conn`）、敏感路径屏蔽、`default_server` 兜底和登录接口加严限流，避免在生产入口"裸奔"。

---

## 2. 部署规划

### 2.1 路径规划


| 类型             | 路径                                          |
| -------------- | ------------------------------------------- |
| 安装根目录          | `/data/module`                              |
| Nginx 版本安装目录   | `/data/module/nginx1.31.1`                  |
| Nginx 稳定软链接    | `/data/module/nginx`                        |
| Nginx 源码包      | `/data/module/nginx-1.31.1.tar.gz`          |
| Nginx 源码解压目录   | `/data/module/nginx1.31.1/src/nginx-1.31.1` |
| Nginx 二进制文件    | `/data/module/nginx/sbin/nginx`             |
| Nginx 主配置文件    | `/data/module/nginx/conf/nginx.conf`        |
| Nginx 站点配置目录   | `/data/module/nginx/conf/conf.d`            |
| Nginx TLS 证书目录 | `/data/module/nginx/conf/certs`             |
| Nginx 日志目录     | `/data/module/nginx/logs`                   |
| Nginx PID 目录   | `/data/module/nginx/run`                    |
| Nginx 临时文件目录   | `/data/module/nginx/temp`                   |
| 前端应用根目录        | `/data/soft/www`                            |
| 示例 Vue 应用目录    | `/data/soft/www/vue-app`                    |
| 示例 Vue 当前版本软链接 | `/data/soft/www/vue-app/current`            |
| 示例 Vue 版本发布目录  | `/data/soft/www/vue-app/releases`           |
| 示例 Vue 回滚备份目录  | `/data/soft/www/vue-app/backup`             |


说明：

- `/data/module/nginx1.31.1` 用于保存 Nginx 软件安装结果；
- `/data/module/nginx` 是稳定软链接，systemd、运维脚本和巡检命令统一使用该路径；
- `/data/module/nginx/conf/conf.d` 存放业务站点配置，主配置文件只保留全局配置；
- `/data/soft/www` 存放前端构建产物，不与 Nginx 安装目录混放；
- `/data` 是本文示例的数据盘挂载点，实际环境如使用 `/data1`、`/mnt/data`、`/u01`，应按现场标准统一替换；
- 推荐采用 `releases + current` 软链接方式发布前端，便于快速回滚。

### 2.2 端口与网络规划


| 项目        | 默认值                      | 说明                                  |
| --------- | ------------------------ | ----------------------------------- |
| HTTP 端口   | `8088`                   | 内网监听，仅用于 301 跳转 HTTPS，不直接对外提供业务     |
| HTTPS 端口  | `8443`                   | 内网监听的业务入口；公网通过外层防火墙 NAT 映射到此端口      |
| 后端 API 地址 | `http://127.0.0.1:8080`  | 示例值，按实际后端服务替换；后端不得与 Nginx 端口冲突      |
| 监听地址      | `0.0.0.0` 或指定内网 IP       | 公网入口按需监听，内网系统建议绑定内网 IP              |
| 公网映射策略    | 外层防火墙 NAT 公网端口 → 内网 8443 | Nginx 自身不直接监听 80/443，公网端口由网络/运维统一规划 |
| 防火墙来源     | 用户网段、负载均衡、WAF、CDN 回源地址   | 不建议无条件对所有来源开放管理类入口                  |


端口选择说明：

- 不使用 `80/443` 的根本原因不是"换端口就安全"——端口号本身不构成任何安全屏障，扫描器对全端口扫描早已是常态。真正的收益是：① 大幅减少 80/443 的无差别蠕虫扫描噪声，让 access.log 更清晰；② 避免与系统中其他可能占用 80/443 的 Web 中间件（如 httpd、Docker、Podman）冲突；③ 监听 ≥1024 的端口为后续 worker/master 直接绑定非特权端口预留空间。
- 不使用 `8080` 作为 HTTP 跳转端口，是因为本文示例后端 API 默认监听 `127.0.0.1:8080`，避免占用冲突，因此选择 `8088`。
- 公网端口由外层防火墙规则决定（可能是 80、443、或任意端口），Nginx 手册不固定。

访问入口（`server_name`）规划：

- 业务 `server_name` 必须**同时包含域名和服务器 IP**，例如 `server_name example.com 203.0.113.10 192.168.1.100;`，目的是同时允许"域名访问"和"裸 IP 访问"两种入口。
- `server_name` 中 IP 的写法是字面量，**不需要带端口号**；Nginx 是根据 HTTP `Host` 头里的主机名部分匹配的。
- `default_server` 兜底块仍然保留，未匹配到上述任一域名或 IP 的请求（如扫描器随便填的 Host）会被 444 直接掐断。
- HTTPS 证书是绑定域名签发的，**用 IP 直接访问 HTTPS 入口浏览器会报证书告警**（CN/SAN 不匹配），这是浏览器行为，不是 Nginx 配置问题；POC 阶段建议 IP 访问优先走 HTTP `8088`，或在 curl 测试时使用 `-k` 跳过校验。生产对外发布只通过域名，不要通过 IP 访问 HTTPS。

### 2.3 账号规划


| 账号               | 用途                            | 权限原则                                                                                            |
| ---------------- | ----------------------------- | ----------------------------------------------------------------------------------------------- |
| Linux 用户 `nginx` | 运行 Nginx worker 进程            | `/sbin/nologin`，不可登录                                                                            |
| Linux 用户 `root`  | 启动 master 进程，由 systemd 管理生命周期 | 仅用于 systemd 管理，不运行 worker；监听 8088/8443 非特权端口理论上可改为 nginx 用户启动 master，本文保留 root master 以兼容现有运维习惯 |
| 发布用户             | 上传 Vue 构建产物                   | 由企业发布平台或运维规范决定，不建议直接复用 `nginx` 用户登录                                                             |


---

## 3. 部署前检查

### 3.1 服务器资源建议


| 资源   | 生产建议                     |
| ---- | ------------------------ |
| CPU  | 2 核及以上，较高并发建议 4 核及以上     |
| 内存   | 2 GB 及以上，较高并发建议 4 GB 及以上 |
| 数据盘  | 独立数据盘，日志量较大时应单独规划空间      |
| 文件系统 | xfs 或 ext4               |
| 网络   | 根据业务入口规划公网、内网或负载均衡回源网络   |
| 时间同步 | 必须启用 chrony 或 ntpd       |


检查基础信息：

```bash
hostnamectl
lscpu
free -h
lsblk
df -h
ip addr
timedatectl
```

### 3.2 系统初始化要求

安装 Nginx 前，服务器应已完成 `CentOS7.9生产环境初始化手册.md` 中的基础初始化。至少包括：

- 时间同步；
- 主机名与 DNS；
- 数据盘挂载；
- 防火墙基线；
- SSH 安全基线；
- 日志与审计基础配置；
- 软件源可用性或离线 RPM 包准备。

### 3.3 检查端口占用

```bash
ss -lntup | egrep ':8088|:8443|:8080'
```

如果 8088、8443 已被其他进程占用，应先确认该进程用途，不得直接强制停止未知生产进程。如果服务器上原本运行了 httpd、其他 Web 中间件或上一版 Nginx 占用了 80/443，迁移到本规划端口后可一并下线，但务必先与业务确认。

### 3.4 检查 SELinux 状态

```bash
getenforce
```

本文默认按企业常见的 `disabled` 或 `permissive` 环境编写。如果企业要求启用 SELinux，需要额外配置 Web 目录、日志目录、反向代理网络访问等 SELinux 策略，不建议在不了解策略的情况下直接上线。

---

## 4. 安装方案选择

CentOS 7.9 下安装 Nginx 常见有三种方案。

### 4.1 方案 A：YUM 安装发行版或第三方仓库 Nginx

优点：

- 安装简单；
- 升级方便；
- systemd、日志目录等默认集成较好。

缺点：

- 版本可能偏旧；
- 编译参数不可控；
- 企业离线环境依赖仓库治理；
- 不同服务器安装结果可能不一致。

适合：普通内网系统、非强管控环境、已有企业标准 YUM 仓库的团队。

### 4.2 方案 B：官方预编译包安装

优点：

- 接近官方默认行为；
- 不需要本机编译；
- 安装速度快。

缺点：

- 依赖外网或企业制品库同步；
- 目录布局与企业自定义规范可能不一致；
- 对 OpenSSL、模块参数控制有限。

适合：能稳定使用官方仓库、且接受官方目录布局的团队。

### 4.3 方案 C：源码编译安装

优点：

- 版本、模块、路径、编译参数可控；
- 适合离线交付和企业标准化；
- 可以统一安装目录、配置目录、日志目录和 systemd 服务；
- 便于后续多版本并存和软链接切换。

缺点：

- 初次部署步骤较多；
- 后续升级需要重新编译；
- 需要维护源码包校验和编译参数。

适合：生产标准化部署、离线环境、需要统一文档和可重复交付的企业团队。

**推荐：** 本文采用 **方案 C：源码编译安装 Nginx 1.31.1 Mainline**。原因是它更适合企业生产标准化、离线部署、目录统一、版本可控和后续回滚。

---

## 5. 源码包准备与校验

企业内网服务器通常不能直接访问外网，不要在生产服务器上直接使用 `wget` 从互联网下载源码。

应提前通过企业制品库、堡垒机文件上传、SFTP、SCP、`rz` 等方式，将以下文件上传到 `/data/module`：

```text
nginx-1.31.1.tar.gz
```

如企业要求全量离线安装，还需要提前准备依赖 RPM 包，或配置可用的内网 YUM 仓库。

创建目录并检查源码包：

```bash
mkdir -p /data/module
cd /data/module
ls -lh nginx-1.31.1.tar.gz
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
sha256sum nginx-1.31.1.tar.gz
```

校验值不一致时不要继续安装，应重新获取可信源码包。

---

## 6. 编译安装 Nginx

### 6.1 安装编译依赖

```bash
yum install -y \
  gcc gcc-c++ make \
  pcre pcre-devel \
  zlib zlib-devel \
  openssl openssl-devel \
  wget tar vim lrzsz net-tools
```

依赖说明：


| 依赖                                     | 用途                                    |
| -------------------------------------- | ------------------------------------- |
| `gcc`、`gcc-c++`、`make`                 | 编译 Nginx                              |
| `pcre`、`pcre-devel`                    | 支持 `location` 正则匹配和 rewrite           |
| `zlib`、`zlib-devel`                    | 支持 gzip 压缩                            |
| `openssl`、`openssl-devel`              | 支持 HTTPS/TLS，stream SSL 模块也依赖 OpenSSL |
| `wget`、`tar`、`vim`、`lrzsz`、`net-tools` | 源码包处理、文件上传、基础编辑和网络排查工具                |


CentOS 7.9 系统自带 OpenSSL 通常为 1.0.2 系列，满足大多数 HTTPS 场景，但不支持 TLS 1.3。如果企业明确要求 TLS 1.3，应单独规划 OpenSSL 新版本静态编译方案或使用经企业验证的新版 Nginx 包。本文为了控制复杂度，默认使用系统 OpenSSL，启用 TLS 1.2。

### 6.2 解压源码包

```bash
cd /data/module
mkdir -p /data/module/nginx1.31.1/src
tar -zxvf nginx-1.31.1.tar.gz -C /data/module/nginx1.31.1/src
cd /data/module/nginx1.31.1/src/nginx-1.31.1
```

### 6.3 配置编译参数

```bash
cd /data/module/nginx1.31.1/src/nginx-1.31.1

./configure \
  --prefix=/data/module/nginx1.31.1 \
  --sbin-path=/data/module/nginx1.31.1/sbin/nginx \
  --conf-path=/data/module/nginx1.31.1/conf/nginx.conf \
  --pid-path=/data/module/nginx1.31.1/run/nginx.pid \
  --lock-path=/data/module/nginx1.31.1/run/nginx.lock \
  --error-log-path=/data/module/nginx1.31.1/logs/error.log \
  --http-log-path=/data/module/nginx1.31.1/logs/access.log \
  --http-client-body-temp-path=/data/module/nginx1.31.1/temp/client_body_temp \
  --http-proxy-temp-path=/data/module/nginx1.31.1/temp/proxy_temp \
  --http-fastcgi-temp-path=/data/module/nginx1.31.1/temp/fastcgi_temp \
  --http-uwsgi-temp-path=/data/module/nginx1.31.1/temp/uwsgi_temp \
  --http-scgi-temp-path=/data/module/nginx1.31.1/temp/scgi_temp \
  --user=nginx \
  --group=nginx \
  --with-compat \
  --with-file-aio \
  --with-threads \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_gzip_static_module \
  --with-http_stub_status_module \
  --with-http_realip_module \
  --with-http_auth_request_module \
  --with-http_slice_module \
  --with-http_sub_module \
  --with-http_addition_module \
  --with-http_secure_link_module \
  --with-http_degradation_module \
  --with-http_flv_module \
  --with-http_mp4_module \
  --with-stream \
  --with-stream_ssl_module \
  --with-stream_realip_module \
  --with-stream_ssl_preread_module
```

编译参数说明：


| 参数                                 | 说明                                    |
| ---------------------------------- | ------------------------------------- |
| `--prefix`                         | Nginx 安装目录                            |
| `--user`、`--group`                 | worker 进程运行用户和用户组                     |
| `--with-http_ssl_module`           | 支持 HTTPS                              |
| `--with-http_v2_module`            | 支持 HTTP/2                             |
| `--with-http_gzip_static_module`   | 支持预压缩 `.gz` 静态文件                      |
| `--with-http_stub_status_module`   | 支持基础状态页，用于监控                          |
| `--with-http_realip_module`        | 支持从负载均衡、WAF、CDN 获取真实客户端 IP            |
| `--with-threads`                   | 支持线程池，提高部分文件 IO 场景能力                  |
| `--with-file-aio`                  | 支持异步文件 IO                             |
| `--with-stream`                    | 支持 TCP/UDP 四层代理能力                     |
| `--with-stream_ssl_module`         | 支持 stream 层 TLS 代理                    |
| `--with-stream_realip_module`      | 支持 stream 层真实客户端 IP 处理                |
| `--with-stream_ssl_preread_module` | 支持在不解密 TLS 的情况下读取 SNI 等信息，适合 TLS 透传分流 |


### 6.4 编译与安装

```bash
# 编译
make -j$(nproc)
# 安装
make install
```

创建稳定软链接：

```bash
ln -sfn /data/module/nginx1.31.1 /data/module/nginx
```

### 6.5 验证安装结果

```bash
/data/module/nginx/sbin/nginx -v
/data/module/nginx/sbin/nginx -V
ls -lh /data/module/nginx/sbin/nginx
```

示例输出：

```text
nginx version: nginx/1.31.1
```

---

## 7. 配置环境变量

Nginx 通过源码编译安装到 `/data/module/nginx1.31.1` 并通过稳定软链接 `/data/module/nginx` 使用。为方便在任意路径直接调用 `nginx` 命令，统一将自定义环境变量维护到 `/etc/profile.d/myenv.sh`，避免为单个软件单独创建脚本文件。

### 7.1 写入环境变量

```bash
cat >> /etc/profile.d/myenv.sh << 'EOF'

# Nginx
export NGINX_HOME=/data/module/nginx
export PATH=$NGINX_HOME/sbin:$PATH
EOF

chmod 644 /etc/profile.d/myenv.sh
```

说明：

- 使用 `>>` 追加，保留 `myenv.sh` 中其他软件（如 MySQL、Redis）已写入的环境变量；
- `NGINX_HOME` 指向稳定软链接 `/data/module/nginx`，升级版本时只需切换软链接，环境变量无需修改；
- `PATH` 优先级中 `NGINX_HOME/sbin` 放在最前，避免被系统其他位置可能存在的 `nginx` 命令覆盖。

### 7.2 加载并验证

```bash
source /etc/profile.d/myenv.sh

echo $NGINX_HOME
which nginx
nginx -v
```

预期 `which nginx` 输出为 `/data/module/nginx/sbin/nginx`，`nginx -v` 输出版本号 `nginx/1.31.1`。新登录的会话会自动加载 `myenv.sh`，无需每次手动 `source`。

---

## 8. OS 内核与资源限制配置

### 8.1 配置 sysctl 参数

不要直接反复向 `/etc/sysctl.conf` 追加配置。生产建议使用独立配置文件，便于维护和回滚。

```bash
cat > /etc/sysctl.d/99-nginx.conf << 'EOF'
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 10240 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
fs.file-max = 2097152
EOF

sysctl --system
```

参数说明：


| 参数                             | 建议值           | 说明                         |
| ------------------------------ | ------------- | -------------------------- |
| `net.core.somaxconn`           | `65535`       | 提高 TCP accept 队列上限         |
| `net.core.netdev_max_backlog`  | `65535`       | 提高网卡接收队列积压上限               |
| `net.ipv4.tcp_max_syn_backlog` | `65535`       | 提高 SYN 队列上限                |
| `net.ipv4.ip_local_port_range` | `10240 65535` | 扩大本机临时端口范围，适合反向代理场景        |
| `net.ipv4.tcp_fin_timeout`     | `15`          | 缩短 FIN-WAIT-2 超时时间         |
| `net.ipv4.tcp_tw_reuse`        | `1`           | 允许复用 TIME-WAIT 连接，用于出站连接场景 |
| `net.ipv4.tcp_keepalive_time`  | `300`         | TCP keepalive 探测时间         |
| `fs.file-max`                  | `2097152`     | 提高系统最大文件句柄数                |


验证：

```bash
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv4.ip_local_port_range
sysctl fs.file-max
```

### 8.2 文件句柄数限制

Nginx 高并发连接依赖文件描述符。生产环境必须提高文件句柄限制。

```bash
cat > /etc/security/limits.d/99-nginx.conf << 'EOF'
nginx soft nofile 200000
nginx hard nofile 200000
root soft nofile 200000
root hard nofile 200000
EOF
```

systemd 服务中也需要设置 `LimitNOFILE=200000`，见 [13. systemd 服务配置](#13-systemd-服务配置)。

---

## 9. 目录、用户与权限

### 9.1 创建 nginx 用户

```bash
id nginx || useradd -r -s /sbin/nologin nginx
```

### 9.2 创建运行目录

```bash
mkdir -p /data/module/nginx/conf/conf.d
mkdir -p /data/module/nginx/conf/certs
mkdir -p /data/module/nginx/logs
mkdir -p /data/module/nginx/run
mkdir -p /data/module/nginx/temp/client_body_temp
mkdir -p /data/module/nginx/temp/proxy_temp
mkdir -p /data/module/nginx/temp/fastcgi_temp
mkdir -p /data/module/nginx/temp/uwsgi_temp
mkdir -p /data/module/nginx/temp/scgi_temp
mkdir -p /data/soft/www/vue-app/releases
mkdir -p /data/soft/www/vue-app/backup
```

目录用途：


| 目录                                | 用途              |
| --------------------------------- | --------------- |
| `/data/module/nginx/conf`         | Nginx 主配置和全局配置  |
| `/data/module/nginx/conf/conf.d`  | 站点配置            |
| `/data/module/nginx/conf/certs`   | TLS 证书和私钥       |
| `/data/module/nginx/logs`         | 访问日志和错误日志       |
| `/data/module/nginx/run`          | PID、lock 等运行时文件 |
| `/data/module/nginx/temp`         | 代理、上传、临时响应文件    |
| `/data/soft/www/vue-app/releases` | 前端版本发布目录        |
| `/data/soft/www/vue-app/current`  | 当前生效前端版本软链接     |
| `/data/soft/www/vue-app/backup`   | 前端回滚备份目录        |


### 9.3 设置权限

```bash
chown -R root:root /data/module/nginx
chown -R nginx:nginx /data/module/nginx/logs /data/module/nginx/run /data/module/nginx/temp
chown -R nginx:nginx /data/soft/www

chmod 755 /data/module/nginx
chmod 750 /data/module/nginx/conf
chmod 750 /data/module/nginx/conf/conf.d
chmod 700 /data/module/nginx/conf/certs
chmod 750 /data/module/nginx/logs /data/module/nginx/run /data/module/nginx/temp
chmod 755 /data/soft/www
chmod 755 /data/soft/www/vue-app
```

证书私钥权限建议：

```bash
chmod 600 /data/module/nginx/conf/certs/*.key 2>/dev/null || true
chmod 644 /data/module/nginx/conf/certs/*.crt 2>/dev/null || true
```

---

## 10. Nginx 标准化配置文件写入

### 10.1 备份默认配置

```bash
cp /data/module/nginx/conf/nginx.conf /data/module/nginx/conf/nginx.conf.bak.$(date +%F-%H%M%S)
```

### 10.2 写入主配置文件

```bash
cat > /data/module/nginx/conf/nginx.conf << 'EOF'
user  nginx nginx;
worker_processes  auto;
worker_rlimit_nofile 200000;

pid   /data/module/nginx/run/nginx.pid;
error_log  /data/module/nginx/logs/error.log warn;

events {
    use epoll;
    worker_connections  65535;
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    charset utf-8;
    server_tokens off;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'uht=$upstream_header_time urt=$upstream_response_time '
                    'upstream="$upstream_addr"';

    access_log  /data/module/nginx/logs/access.log  main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 65;
    keepalive_requests 10000;

    client_header_timeout 15s;
    client_body_timeout 60s;
    send_timeout 60s;

    client_max_body_size 50m;
    client_body_buffer_size 256k;
    large_client_header_buffers 4 16k;

    open_file_cache max=10000 inactive=60s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_vary on;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/json
        application/xml
        application/rss+xml
        image/svg+xml;

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    # ---------- 公网入口基础防护：限流共享内存区 ----------
    # 这里只"定义"共享内存区，不触发限流；真正生效由各业务 location
    # 内的 limit_req / limit_conn 指令显式引用对应 zone。
    # 1) per-IP 页面入口限流：用于 location / （SPA 入口/未匹配路径）
    limit_req_zone  $binary_remote_addr zone=perip_req:10m  rate=20r/s;
    # 2) /api/ 路径限流
    limit_req_zone  $binary_remote_addr zone=api_req:10m    rate=10r/s;
    # 3) 登录/敏感接口限流（防爆破）
    limit_req_zone  $binary_remote_addr zone=login_req:10m  rate=5r/m;
    # 4) per-IP 并发连接数（在 server 级引用，覆盖整站）
    limit_conn_zone $binary_remote_addr zone=perip_conn:10m;
    # 超限响应码：429 比默认的 503 更符合语义，监控告警识别更直观
    limit_req_status  429;
    limit_conn_status 429;
    # ----------------------------------------------------

    include /data/module/nginx/conf/conf.d/*.conf;
}
EOF
```

### 10.3 主配置说明


| 配置项                           | 说明                                     |
| ----------------------------- | -------------------------------------- |
| `worker_processes auto`       | 根据 CPU 自动设置 worker 数量                  |
| `worker_connections 65535`    | 单 worker 最大连接数上限                       |
| `worker_rlimit_nofile 200000` | Nginx 进程文件句柄上限                         |
| `server_tokens off`           | 隐藏 Nginx 版本号，减少信息暴露                    |
| `gzip on`                     | 开启 gzip 压缩，降低静态资源传输体积                  |
| `open_file_cache`             | 缓存静态文件元信息，提高静态资源访问效率                   |
| `log_format main`             | 增加 upstream 耗时，便于排查后端慢请求               |
| `map $http_upgrade`           | 为 WebSocket 代理预留标准变量                   |
| `limit_req_zone perip_req`    | 页面入口限流共享内存区，20r/s（由 `location /` 引用）  |
| `limit_req_zone api_req`      | `/api/` 路径限流共享内存区，10r/s                |
| `limit_req_zone login_req`    | 登录接口限流共享内存区，5r/min（防爆破）                |
| `limit_conn_zone perip_conn`  | 每 IP 并发连接数共享内存区（由 server 级引用，覆盖整站）     |
| `limit_req_status 429`        | 限流命中返回 `429 Too Many Requests`，监控告警更直观 |


### 10.4 公网入口防护配置说明

#### 10.4.1 主配置与站点配置的协同关系

本手册采用“**主配置统一定义限流/连接统计区，站点配置按路径显式引用**”的组织方式。配置人员需要先理解：**第 10.2 节写入的 `limit_req_zone`、`limit_conn_zone` 是后续所有业务站点限流配置的前置依赖**；第 11、12 章站点配置中的 `limit_req zone=...`、`limit_conn ...` 都不是孤立配置，必须依赖主配置中已经定义好的同名 zone。

| 配置位置 | 指令 | 作用 | 是否直接限流 |
| --- | --- | --- | --- |
| `/data/module/nginx/conf/nginx.conf` 的 `http {}` | `limit_req_zone` | 定义请求频率限流共享内存区，例如 `perip_req`、`api_req`、`login_req` | 否 |
| `/data/module/nginx/conf/nginx.conf` 的 `http {}` | `limit_conn_zone` | 定义并发连接数统计共享内存区，例如 `perip_conn` | 否 |
| `/data/module/nginx/conf/conf.d/*.conf` 的 `server` / `location` | `limit_req` | 引用某个 `limit_req_zone`，对当前路径启用请求频率限流 | 是 |
| `/data/module/nginx/conf/conf.d/*.conf` 的 `server` / `location` | `limit_conn` | 引用某个 `limit_conn_zone`，对当前范围启用并发连接数限制 | 是 |

也就是说，主配置里的 `limit_req_zone` / `limit_conn_zone` 只是“先把规则池建好”，不会单独拦截任何请求；只有业务站点配置中写了 `limit_req` / `limit_conn`，引用对应 zone 后才真正生效。

示例对应关系如下：

```nginx
# nginx.conf：定义请求限流共享内存区
limit_req_zone  $binary_remote_addr zone=perip_req:10m  rate=20r/s;
limit_req_zone  $binary_remote_addr zone=api_req:10m    rate=10r/s;
limit_req_zone  $binary_remote_addr zone=login_req:10m  rate=5r/m;
limit_conn_zone $binary_remote_addr zone=perip_conn:10m;

# conf.d/业务站点.conf：按路径引用不同 zone
server {
    limit_conn perip_conn 200;

    location = /api/auth/login {
        limit_req zone=login_req burst=3 nodelay;
        proxy_pass http://backend;
    }

    location /api/ {
        limit_req zone=api_req burst=20 nodelay;
        proxy_pass http://backend;
    }

    location / {
        limit_req zone=perip_req burst=40 nodelay;
        try_files $uri $uri/ /index.html;
    }
}
```

配置人员应按以下规则维护：

1. **主配置只定义通用 zone**：所有业务站点共用的 `limit_req_zone`、`limit_conn_zone` 统一放在 `nginx.conf` 的 `http {}` 内；
2. **站点配置只引用 zone**：`conf.d/*.conf` 中不要重复定义 `limit_req_zone` / `limit_conn_zone`，只在需要限流的 `server` 或 `location` 中引用；
3. **不同路径引用不同策略**：页面入口用 `perip_req`，普通 API 用 `api_req`，登录等敏感接口用 `login_req`；
4. **静态资源不引用 `limit_req`**：`.js`、`.css`、图片、字体等静态资源由静态资源 location 直接处理，不写 `limit_req`，避免 SPA 首屏资源并发加载时误触发 `429`；
5. **连接数限制可以放在 server 级**：`limit_conn perip_conn 200` 限制的是同一 IP 的并发连接数，通常可以放在业务 `server` 级覆盖整站；
6. **请求频率限制不要放在 server 级**：`limit_req` 放在 `server` 级会继承到静态资源等所有 location，容易误伤前端资源加载，应放到具体需要保护的 location 中。

#### 10.4.2 限流策略说明

主配置中的 `limit_req_zone`/`limit_conn_zone` 只是**定义共享内存区**，本身不触发限流。真正的限流由站点配置（第 11、12 章）中的 `limit_req`、`limit_conn` 指令引用共享内存区后生效。

本手册采用 **"按 location 显式启用限流"** 的策略：

- `limit_req` 不在 `server` 级声明，而是分别在 `location /`（页面入口/SPA fallback）、`location /api/`、`location = /api/auth/login` 内显式启用，引用对应 zone；
- 静态资源 location（`*.js`、`*.css`、字体、图片等）**不写** `limit_req`，自然就不参与限流——SPA 首屏并发加载几十甚至上百个带 hash 的 chunk 与 assets 不会被误伤；
- `limit_conn perip_conn` 在 `server` 级声明一次，覆盖全站，留出 HTTP/2 多路复用和图片并行下载所需空间。

> 不要在某个 location 里写 `limit_req off;`：Nginx 没有这个语法，`nginx -t` 会直接报 `invalid parameter "off"`。如果要"关闭某个 location 的限流"，正确做法是在 `server` 级别不声明 `limit_req`，由具体需要限流的 location 自行声明。

阈值选取原则：

- **POC 与中小业务的宽松保守值**：页面入口 20r/s + burst 40，足以应对正常用户和扫描器混合流量，不会误伤业务；
- `**/api/` 接口**：10r/s + burst 20，覆盖一般前端调用峰值；
- `**/api/auth/login` 等登录类接口**：5 次/分钟，专门针对密码爆破和验证码穷举。正常用户登录一分钟内尝试不会超过 2-3 次；
- **并发连接 `perip_conn 200`**：面向公网企业客户时，多个用户可能共用同一个公网出口 IP，200 能更好兼容 NAT 后的多人访问、keep-alive、HTTP/2 多路复用和 WebSocket 连接。

阈值校准方法：

- 上线后观察一周 `error.log` 中 `limiting requests` 和 `limiting connections` 出现频率；
- 如果**正常业务**频繁被限，按 1.5 ~ 2 倍逐步放宽；
- 如果**爬虫/扫描**仍能正常打到后端，按 0.5 ~ 0.7 倍逐步收紧；
- 不要一开始就把阈值设得过紧或过松。

不应该在单体 Nginx 上加的（避免过度设计）：

- ModSecurity / Naxsi 等 WAF 模块：规则维护成本远高于单体场景收益，误杀率高；
- 大型 IP 黑名单或 GeoIP 国别封锁：除非业务明确只面向特定国家；
- 复杂 User-Agent 黑名单：扫描器轮换 UA 成本极低，且容易误杀搜索引擎、监控、健康检查；
- fail2ban 联动封禁：单体 + 低流量场景规则维护负担过重，公网映射端口本身已大幅降噪。

> **静态资源长缓存头不要加 `always`**：Nginx 默认 `add_header` 只对 `200/201/204/206/301/302/303/304/307/308` 生效，错误响应（4xx/5xx）不会被打上响应头。如果给静态资源 `Cache-Control: public, max-age=2592000, immutable` 加了 `always`，那么万一限流（其他 location）或上游异常返回 4xx/5xx，错误响应也会被打上 30 天 immutable 缓存头，被浏览器/CDN 长期缓存，问题被永久放大。所以**静态资源的长缓存头不加 `always`**；其他需要在错误页也下发的安全响应头（如 `X-Content-Type-Options`、`X-Frame-Options`、`HSTS`）才用 `always`。

---

## 11. Vue 前端站点配置

### 11.1 准备示例前端目录

实际生产中，`dist` 目录应由 CI/CD 或发布人员上传。本文先准备一个最小示例页面，用于验证 Nginx 是否正常工作。

```bash
mkdir -p /data/soft/www/vue-app/releases/20260616_000001
cat > /data/soft/www/vue-app/releases/20260616_000001/index.html << 'EOF'
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Nginx Vue App</title>
</head>
<body>
  <h1>Nginx Vue App is running</h1>
</body>
</html>
EOF
ln -sfn /data/soft/www/vue-app/releases/20260616_000001 /data/soft/www/vue-app/current
chown -R nginx:nginx /data/soft/www/vue-app
```

真实 Vue 项目构建示例：

```bash
npm ci
npm run build
```

构建完成后，将 `dist` 目录内容发布到类似目录：

```text
/data/soft/www/vue-app/releases/20260616_120000/
```

### 11.2 写入 HTTP 站点配置

如果暂时没有 HTTPS 证书，可先使用 HTTP 配置完成基础部署验证。本节同时演示 `default_server` 兜底、限流、敏感路径屏蔽等公网入口必备防护，配置整体不长，但缺一不可。

> **限流依赖说明**：本节站点配置中的 `limit_conn perip_conn 200`、`limit_req zone=login_req ...`、`limit_req zone=api_req ...`、`limit_req zone=perip_req ...` 都依赖第 10.2 节主配置 `nginx.conf` 中已经定义的 `limit_conn_zone` / `limit_req_zone`。如果未先写入主配置中的 zone 定义，`nginx -t` 会因找不到对应 zone 而失败。

```bash
cat > /data/module/nginx/conf/conf.d/vue-app.conf << 'EOF'
upstream vue_app_backend {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# ---------- default_server 兜底 ----------
# 拒绝裸 IP / 未知 Host 访问，扫描器收到 444 直接断连
server {
    listen 8088 default_server;
    server_name _;
    return 444;
}
# -----------------------------------------

server {
    listen 8088;
    server_name example.com <公网IP> <内网IP>;
    # ↑ 同时允许域名访问和裸 IP 访问。将占位符替换为现场真实 IP，
    #   例如：server_name example.com 203.0.113.10 192.168.1.100;
    #   未列入的 Host（含其他乱填 Host）会被上面的 default_server 兜底块掐断。

    root /data/soft/www/vue-app/current;
    index index.html;

    access_log /data/module/nginx/logs/vue-app-access.log main;
    error_log  /data/module/nginx/logs/vue-app-error.log warn;

    # ---------- 基础防护 ----------
    limit_conn perip_conn 200;

    # 敏感路径屏蔽：拒绝访问 .git/.env/备份文件等
    location ~ /\.(git|svn|hg|env|DS_Store) { deny all; return 404; }
    location ~* \.(bak|sql|swp|old|orig|save|log)$ { deny all; return 404; }
    # ------------------------------

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location = /favicon.ico {
        access_log off;
        log_not_found off;
        try_files $uri =204;
    }

    location = /robots.txt {
        access_log off;
        log_not_found off;
        try_files $uri =404;
    }

    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate" always;
        add_header Pragma "no-cache" always;
        add_header Expires "0" always;
        try_files $uri =404;
    }

    location ~* \.(?:js|css|png|jpg|jpeg|gif|ico|svg|webp|woff|woff2|ttf|eot)$ {
        # 静态资源不参与限流：server 级不声明 limit_req，本 location 也不声明，
        # SPA 首屏并发加载的几十上百个 chunk 和 assets 不会被误伤 429。
        # 同时不使用 always，避免错误响应被浏览器按 immutable 长缓存。
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
        try_files $uri =404;
    }

    # 登录类敏感接口：单独加严限流（防爆破）
    location = /api/auth/login {
        rewrite ^/api/(.*)$ /$1 break;
        limit_req zone=login_req burst=3 nodelay;

        proxy_pass http://vue_app_backend;
        proxy_http_version 1.1;

        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers 8 64k;
        proxy_busy_buffers_size 128k;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }

    location /api/ {
        rewrite ^/api/(.*)$ /$1 break;
        limit_req zone=api_req burst=20 nodelay;

        proxy_pass http://vue_app_backend;
        proxy_http_version 1.1;

        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers 8 64k;
        proxy_busy_buffers_size 128k;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }

    location /ws/ {
        proxy_pass http://vue_app_backend;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location /nginx_status {
        stub_status;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }

    location / {
        limit_req zone=perip_req burst=40 nodelay;
        try_files $uri $uri/ /index.html;
    }
}
EOF
```

需要将 `server_name example.com` 替换为真实域名或内网访问域名。如果没有域名，可临时使用服务器 IP 访问，但生产环境建议使用域名。
此外 `server_name` 中的 `<公网IP>`、`<内网IP>` 占位符也必须替换为现场真实 IP，可通过 `ip addr` 在服务器上查询，公网映射 IP 由网络/运维同事提供。如果只想用域名访问，删掉两个占位符即可。

> 注：本节示例仅为"HTTP 端口（8088）已能独立提供业务"的最小可运行配置，用于无证书阶段的部署验证。**生产公网入口必须使用 HTTPS，请直接采用第 12 章 HTTPS 配置**，届时 8088 端口的角色会变成"301 跳转 HTTPS"，不再独立承载业务。

### 11.3 API 路径转发规则说明

`proxy_pass` 是否带尾部 `/`、是否配合 `rewrite` 会影响后端收到的路径，必须按后端实际接口统一确认。

#### 方案 A：保留 `/api/` 前缀（后端接口路径自带 `/api`）

```nginx
location /api/ {
    proxy_pass http://vue_app_backend;
}
```

请求路径：

```text
/api/user/list -> 后端收到 /api/user/list
```

#### 方案 B：通过 `proxy_pass` 尾部斜杠去掉 `/api/` 前缀

```nginx
location /api/ {
    proxy_pass http://vue_app_backend/;
}
```

请求路径：

```text
/api/user/list -> 后端收到 /user/list
```

#### 方案 C：使用 `rewrite` 显式改写（**本文档采用**）

```nginx
location /api/ {
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://vue_app_backend;
}
```

请求路径：

```text
/api/user/list -> 后端收到 /user/list
```

**推荐：** 本文档生产基准统一采用 **方案 C：rewrite 显式剥离 `/api/` 前缀**。原因：

- 后端接口路径**不带 `/api/`**（如 `/user/list`、`/auth/login`），可独立部署、独立测试，`curl http://backend:8080/user/list` 直连即可验证；
- 网关层面统一加 `/api/` 业务前缀，前后端职责清晰、解耦彻底；
- 与方案 B 相比，`rewrite` 行为更显式，配置阅读时 `proxy_pass` 是否带 `/` 不再有歧义，多人维护更不易出错；
- 与方案 A 相比，更符合 Spring Cloud Gateway、Kong、APISIX 等主流网关"前缀剥离"的统一做法，未来从单体 Nginx 演进到网关集群迁移成本最低。

如果某个具体接口的后端**确实**自带 `/api/` 前缀，可在 `location` 内移除 `rewrite` 那一行单独保留前缀（属个例处理，应在 PR/上线评审时单独说明）。

---

## 12. HTTPS 证书配置

### 12.1 上传证书文件

将证书文件上传到：

```text
/data/module/nginx/conf/certs/example.com.crt
/data/module/nginx/conf/certs/example.com.key
```

设置权限：

```bash
chown root:root /data/module/nginx/conf/certs/example.com.crt /data/module/nginx/conf/certs/example.com.key
chmod 644 /data/module/nginx/conf/certs/example.com.crt
chmod 600 /data/module/nginx/conf/certs/example.com.key
```

### 12.2 生成 DH 参数文件

```bash
openssl dhparam -out /data/module/nginx/conf/certs/dhparam.pem 2048
chmod 644 /data/module/nginx/conf/certs/dhparam.pem
```

### 12.3 HTTP 跳转 HTTPS 配置

公网生产系统推荐 HTTP（内网监听端口 `8088`）仅用于跳转 HTTPS（内网监听端口 `8443`）。本节同步给出 `default_server` 兜底、限流、敏感路径屏蔽、登录接口加严等全部公网入口必备防护，**这是生产入口的最终形态**。

> **限流依赖说明**：本节 HTTPS 业务入口中的 `limit_conn perip_conn 200`、`limit_req zone=login_req ...`、`limit_req zone=api_req ...`、`limit_req zone=perip_req ...` 都依赖第 10.2 节主配置 `nginx.conf` 中已经定义的 `limit_conn_zone` / `limit_req_zone`。站点配置只负责引用，不应在 `conf.d/*.conf` 中重复定义 zone。

```bash
cat > /data/module/nginx/conf/conf.d/vue-app.conf << 'EOF'
upstream vue_app_backend {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# ---------- default_server 兜底（HTTP 8088）----------
server {
    listen 8088 default_server;
    server_name _;
    return 444;
}

# ---------- default_server 兜底（HTTPS 8443）----------
# 复用同一份证书即可，未知 Host 直接 444
server {
    listen 8443 ssl http2 default_server;
    server_name _;
    ssl_certificate     /data/module/nginx/conf/certs/example.com.crt;
    ssl_certificate_key /data/module/nginx/conf/certs/example.com.key;
    ssl_protocols TLSv1.2;
    return 444;
}

# ---------- HTTP → HTTPS 301 跳转 ----------
server {
    listen 8088;
    server_name example.com <公网IP> <内网IP>;
    # ↑ 域名 + IP 均跳转 HTTPS。$host 会保留访问端使用的主机名，
    #   保证用 IP 访问 HTTP 时也会被跳到对应 IP 的 HTTPS（仅会带证书告警，详见第 21.10）。
    return 301 https://$host:8443$request_uri;
}

# ---------- HTTPS 业务入口 ----------
server {
    listen 8443 ssl http2;
    server_name example.com <公网IP> <内网IP>;
    # ↑ 同时允许域名访问和裸 IP 访问。将占位符替换为现场真实 IP，
    #   例如：server_name example.com 203.0.113.10 192.168.1.100;
    #   未匹配的 Host 会被上面的 default_server 兜底块掐断。
    #   注意：HTTPS 用 IP 访问会触发浏览器证书告警，详见第 21.10。

    root /data/soft/www/vue-app/current;
    index index.html;

    access_log /data/module/nginx/logs/vue-app-access.log main;
    error_log  /data/module/nginx/logs/vue-app-error.log warn;

    ssl_certificate     /data/module/nginx/conf/certs/example.com.crt;
    ssl_certificate_key /data/module/nginx/conf/certs/example.com.key;
    ssl_dhparam         /data/module/nginx/conf/certs/dhparam.pem;

    ssl_protocols TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=15552000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # ---------- 基础防护 ----------
    limit_conn perip_conn 200;

    # 敏感路径屏蔽：拒绝访问 .git/.env/备份文件等
    location ~ /\.(git|svn|hg|env|DS_Store) { deny all; return 404; }
    location ~* \.(bak|sql|swp|old|orig|save|log)$ { deny all; return 404; }
    # ------------------------------

    location = /favicon.ico {
        access_log off;
        log_not_found off;
        try_files $uri =204;
    }

    location = /robots.txt {
        access_log off;
        log_not_found off;
        try_files $uri =404;
    }

    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate" always;
        add_header Pragma "no-cache" always;
        add_header Expires "0" always;
        try_files $uri =404;
    }

    location ~* \.(?:js|css|png|jpg|jpeg|gif|ico|svg|webp|woff|woff2|ttf|eot)$ {
        # 静态资源不参与限流：server 级不声明 limit_req，本 location 也不声明，
        # SPA 首屏并发加载的几十上百个 chunk 和 assets 不会被误伤 429。
        # 同时不使用 always，避免错误响应被浏览器按 immutable 长缓存。
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
        try_files $uri =404;
    }

    # 登录类敏感接口：单独加严限流（5r/min，防爆破）
    # 实际路径请按后端真实接口调整（如 /api/login、/api/user/login 等）
    location = /api/auth/login {
        rewrite ^/api/(.*)$ /$1 break;
        limit_req zone=login_req burst=3 nodelay;

        proxy_pass http://vue_app_backend;
        proxy_http_version 1.1;

        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers 8 64k;
        proxy_busy_buffers_size 128k;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }

    location /api/ {
        rewrite ^/api/(.*)$ /$1 break;
        limit_req zone=api_req burst=20 nodelay;

        proxy_pass http://vue_app_backend;
        proxy_http_version 1.1;

        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers 8 64k;
        proxy_busy_buffers_size 128k;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }

    location /ws/ {
        proxy_pass http://vue_app_backend;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location /nginx_status {
        stub_status;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }

    location / {
        limit_req zone=perip_req burst=40 nodelay;
        try_files $uri $uri/ /index.html;
    }
}
EOF
```

### 12.4 HTTPS 与防护配置说明


| 配置项                                         | 说明                                           |
| ------------------------------------------- | -------------------------------------------- |
| `listen 8443 ssl http2`                     | 启用 HTTPS 和 HTTP/2，使用非特权端口 8443               |
| `listen 8443 ssl ... default_server`        | 兜底拦截裸 IP / 未知 Host 的扫描器请求，返回 444 不回包         |
| `ssl_protocols TLSv1.2`                     | CentOS 7.9 系统 OpenSSL 默认适合 TLS 1.2           |
| `ssl_session_cache`                         | 启用 TLS 会话缓存，减少握手开销                           |
| `ssl_session_tickets off`                   | 降低会话票据密钥管理风险                                 |
| `Strict-Transport-Security`                 | 强制浏览器后续使用 HTTPS，公网确认 HTTPS 稳定后再启用            |
| `limit_conn perip_conn 200`                  | 单 IP 并发连接上限 200，兼容公网企业客户同出口 NAT、HTTP/2 多路复用和 WebSocket 连接 |
| `limit_req zone=perip_req burst=40 nodelay` | 页面入口限流 20r/s，瞬时突发 40 个（在 `location /` 内生效）  |
| `limit_req zone=api_req burst=20 nodelay`   | `/api/` 限流 10r/s，瞬时突发 20 个                   |
| `limit_req zone=login_req burst=3 nodelay`  | `/api/auth/login` 限流 5r/min，专门防密码爆破          |
| 敏感路径屏蔽                                      | `.git`、`.env`、`.bak`、`.sql` 等扫描器最常探测路径直接 404 |


如果使用新版 OpenSSL 编译并确认支持 TLS 1.3，可调整为：

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

> **注：登录接口路径示意为 `/api/auth/login`，请按后端真实接口替换**（常见的还有 `/api/login`、`/api/user/login`、`/api/v1/auth/login` 等）。如果业务有验证码、短信验证码等接口，同样建议挂上 `login_req` 限流区。

---

## 13. systemd 服务配置

### 13.1 写入 systemd 服务文件

```bash
cat > /etc/systemd/system/nginx.service << 'EOF'
[Unit]
Description=Nginx HTTP and reverse proxy server
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/data/module/nginx/run/nginx.pid
ExecStartPre=/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
ExecStart=/data/module/nginx/sbin/nginx -c /data/module/nginx/conf/nginx.conf
ExecReload=/data/module/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
LimitNOFILE=200000
TimeoutStartSec=30
TimeoutStopSec=30
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
```

### 13.2 加载并设置开机自启

```bash
systemctl daemon-reload
systemctl enable nginx
```

### 13.3 校验配置

```bash
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
```

配置校验通过后再启动。

---

## 14. 网络边界与访问控制（可选）

本文不维护本机 `firewalld` 规则的全部细节（主机防火墙基线由《CentOS7.9生产环境初始化手册.md》统一规定），但 Nginx 端口切换到 `8088/8443` 后，本机和外层防火墙都需要做对应调整。

### 14.1 部署原则

- HTTP/HTTPS 端口仅对可信网络开放，不得直接暴露到公网；
- Nginx 内网监听 `8088`（HTTP 跳转）和 `8443`（HTTPS 业务），不监听 `80/443`；
- 公网入口通过外层防火墙、安全组、负载均衡 NAT 映射到内网 `8443`，公网端口由网络/运维统一规划（可以是 80、443 或其他端口）；
- 管理端口和业务端口应分离授权；
- 访问来源、端口用途和开放范围必须由外层防火墙、安全组、ACL、VPN、堡垒机或统一网络策略控制；
- 访问策略变更应记录到资产台账或网络策略记录；
- 如果 Nginx 和后端部署在同机，后端服务建议只监听 `127.0.0.1` 或内网 IP；分机部署时，后端服务器只允许 Nginx 服务器内网 IP 访问后端端口。

### 14.2 本机 firewalld 规则调整（按现场实际执行）

如果服务器使用 firewalld 且原本开放了 80/443，需要按下列步骤切换：

```bash
firewall-cmd --permanent --add-port=8088/tcp
firewall-cmd --permanent --add-port=8443/tcp
firewall-cmd --permanent --remove-service=http   2>/dev/null
firewall-cmd --permanent --remove-service=https  2>/dev/null
firewall-cmd --permanent --remove-port=80/tcp    2>/dev/null
firewall-cmd --permanent --remove-port=443/tcp   2>/dev/null
firewall-cmd --reload
firewall-cmd --list-all
```

如果服务器使用 iptables，则按企业 iptables 规范同步调整放行规则，原则一致。

### 14.3 外层防火墙 NAT 映射调整

由网络/运维同事在公网入口防火墙、负载均衡或安全组上执行：

- 公网 HTTPS 端口（例如 443 或自定义端口） → 内网 `服务器IP:8443`；
- 公网 HTTP 端口（可选，仅用于跳转） → 内网 `服务器IP:8088`；
- 删除旧的 "公网端口 → 80/443" 的 NAT 规则。

> POC 阶段如果不需要 HTTP 跳转入口，可只映射 8443，让公网用户直接走 HTTPS。

---

## 15. 启动与基础验证

### 15.1 启动 Nginx

```bash
systemctl start nginx
systemctl status nginx
```

### 15.2 查看监听端口

```bash
ss -lntup | egrep ':8088|:8443'
```

应只看到 `8088` 和 `8443`，没有 `80/443`。

### 15.3 本机访问验证

HTTP 端口（8088）验证，应返回 301 跳转：

```bash
curl -I http://127.0.0.1:8088/
```

HTTPS 端口（8443）验证：

```bash
curl -k -I https://127.0.0.1:8443/
curl -k https://127.0.0.1:8443/
```

如果配置了域名但 DNS 尚未生效，可使用 `--resolve` 临时验证：

```bash
curl -k --resolve example.com:8443:127.0.0.1 https://example.com:8443/
```

### 15.4 Vue history 路由验证

```bash
curl -k -I https://127.0.0.1:8443/user/list
```

期望返回 `200`，并实际回退到 `index.html`。

### 15.5 API 代理验证

后端服务启动后验证：

```bash
curl -k -I https://127.0.0.1:8443/api/health
```

如后端未启动，通常会返回 `502 Bad Gateway`，此时应检查 upstream 地址和后端服务状态。

### 15.6 基础防护验证

裸 IP / 未知 Host 应被 `default_server` 拦截（返回空响应 / 连接关闭）：

```bash
curl -k -I https://127.0.0.1:8443/ -H "Host: scanner.example.com"
```

敏感路径应返回 404：

```bash
curl -k -I https://127.0.0.1:8443/.git/config
curl -k -I https://127.0.0.1:8443/backup.sql
```

限流验证（短时间高频请求，应出现 429）：

```bash
for i in {1..80}; do curl -ks -o /dev/null -w "%{http_code} " https://127.0.0.1:8443/; done; echo
```

登录接口加严限流验证（连续访问 5 次以上应快速出现 429）：

```bash
for i in {1..10}; do curl -ks -o /dev/null -w "%{http_code} " -X POST https://127.0.0.1:8443/api/auth/login; done; echo
```

### 15.7 状态页验证

```bash
curl http://127.0.0.1:8088/nginx_status   # 8088 默认 server 返回 444，需要带正确 Host 才会进入业务 server
curl -k https://127.0.0.1:8443/nginx_status -H "Host: example.com"
```

示例输出：

```text
Active connections: 1 
server accepts handled requests
 1 1 1 
Reading: 0 Writing: 1 Waiting: 0
```

---

## 16. 生产验证清单

上线前必须完成以下检查。

### 16.1 安装与进程检查

```bash
/data/module/nginx/sbin/nginx -v
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl is-enabled nginx
systemctl status nginx
ps -ef | grep nginx | grep -v grep
```

确认：

- master 进程通常由 `root` 启动；
- worker 进程必须由 `nginx` 用户运行；
- systemd 状态为 `active (running)`；
- 开机自启已启用。

### 16.2 端口检查

```bash
ss -lntup | egrep ':8088|:8443'
```

确认：

- 只监听预期端口（`8088`、`8443`）；
- 没有额外暴露未规划端口；
- 没有遗留的 `80/443` 监听。

### 16.3 前端访问检查

```bash
curl -kI https://127.0.0.1:8443/
curl -kI https://127.0.0.1:8443/index.html
curl -kI https://127.0.0.1:8443/assets/app.js
curl -kI https://127.0.0.1:8443/non-exist-route
curl -I  http://127.0.0.1:8088/
```

确认：

- 首页返回 `200`；
- Vue history 路由刷新返回 `200`；
- 静态资源存在时返回 `200`，不存在时返回 `404`；
- `index.html` 不强缓存；
- 带 hash 的静态资源长缓存；
- `8088` 访问返回 `301` 跳转 `https://...:8443/...`。

### 16.4 HTTPS 检查

```bash
curl -k -I https://127.0.0.1:8443/
openssl s_client -connect 127.0.0.1:8443 -servername example.com </dev/null
```

确认：

- 证书链有效；
- 私钥权限安全；
- HTTP（`8088`）能跳转 HTTPS（`8443`）；
- 只启用符合企业安全要求的 TLS 协议。

### 16.5 日志检查

```bash
ls -lh /data/module/nginx/logs
tail -n 50 /data/module/nginx/logs/error.log
tail -n 50 /data/module/nginx/logs/vue-app-access.log
```

确认：

- 访问日志正常写入；
- 错误日志无持续异常；
- 日志目录权限正确。

### 16.6 安全检查

```bash
curl -kI https://127.0.0.1:8443/
curl -kI https://127.0.0.1:8443/.git/config
curl -kI https://127.0.0.1:8443/backup.sql
curl -kI https://127.0.0.1:8443/nginx_status -H "Host: example.com"
for i in {1..80}; do curl -ks -o /dev/null -w "%{http_code} " https://127.0.0.1:8443/; done; echo
for i in {1..10}; do curl -ks -o /dev/null -w "%{http_code} " -X POST https://127.0.0.1:8443/api/auth/login; done; echo
```

确认：

- 响应头不暴露 Nginx 具体版本；
- 敏感路径（`.git`、`.env`、`.bak`、`.sql` 等）返回 `404`；
- `nginx_status` 只允许本机或监控 IP 访问；
- Web 根目录不可访问证书、配置、备份文件；
- API 后端端口没有对公网直接暴露；
- 高频请求能触发 `429`，登录接口爆破前几次能触发 `429`。

---

## 17. 运维检查命令

### 17.1 常用 systemd 命令

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl enable nginx
systemctl disable nginx
```

### 17.2 Nginx 常用命令

```bash
/data/module/nginx/sbin/nginx -v
/data/module/nginx/sbin/nginx -V
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
/data/module/nginx/sbin/nginx -s reload
/data/module/nginx/sbin/nginx -s quit
```

### 17.3 查看连接与端口

```bash
ss -lntup | grep nginx
ss -ant | awk '{print $1}' | sort | uniq -c
```

### 17.4 查看日志

```bash
tail -f /data/module/nginx/logs/access.log
tail -f /data/module/nginx/logs/error.log
tail -f /data/module/nginx/logs/vue-app-access.log
tail -f /data/module/nginx/logs/vue-app-error.log
```

### 17.5 查看资源占用

```bash
ps -eo pid,ppid,user,cmd,%cpu,%mem --sort=-%cpu | grep nginx
top -p $(pgrep -d',' nginx)
```

### 17.6 统计访问状态码

```bash
awk '{print $9}' /data/module/nginx/logs/vue-app-access.log | sort | uniq -c | sort -nr
```

### 17.7 统计慢请求

```bash
awk '{for(i=1;i<=NF;i++){if($i ~ /^rt=/){split($i,a,"="); if(a[2] > 1) print $0}}}' /data/module/nginx/logs/vue-app-access.log
```

---

## 18. 前端发布、回滚与缓存刷新

### 18.1 推荐发布目录结构

```text
/data/soft/www/vue-app/
├── current -> /data/soft/www/vue-app/releases/20260616_120000
├── releases/
│   ├── 20260615_180000/
│   └── 20260616_120000/
└── backup/
```

### 18.2 发布新版本

假设新的 Vue 构建产物已上传到 `/tmp/dist`。

```bash
release_id=$(date +%Y%m%d_%H%M%S)
release_dir=/data/soft/www/vue-app/releases/${release_id}

mkdir -p ${release_dir}
cp -a /tmp/dist/. ${release_dir}/
chown -R nginx:nginx ${release_dir}
ln -sfn ${release_dir} /data/soft/www/vue-app/current
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl reload nginx
```

### 18.3 验证新版本

```bash
curl -kI https://127.0.0.1:8443/
curl -kI https://127.0.0.1:8443/index.html
readlink -f /data/soft/www/vue-app/current
```

### 18.4 回滚到上一版本

查看版本目录：

```bash
ls -lt /data/soft/www/vue-app/releases
```

切换 `current` 到上一版本：

```bash
ln -sfn /data/soft/www/vue-app/releases/20260615_180000 /data/soft/www/vue-app/current
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl reload nginx
```

验证：

```bash
readlink -f /data/soft/www/vue-app/current
curl -kI https://127.0.0.1:8443/
```

### 18.5 缓存策略说明

Vue 前端生产部署推荐：


| 文件类型            | 推荐缓存策略                                | 原因                      |
| --------------- | ------------------------------------- | ----------------------- |
| `index.html`    | `no-cache, no-store, must-revalidate` | 避免入口文件长期缓存导致用户无法加载新版本   |
| 带 hash 的 JS/CSS | 长缓存                                   | 文件名变化即可更新缓存             |
| 图片、字体           | 根据是否带 hash 决定                         | 带 hash 可长缓存，不带 hash 应谨慎 |
| API 响应          | 默认不由 Nginx 缓存                         | 避免业务数据一致性问题             |


如果前面还有 CDN，发布后还需要按 CDN 平台规则刷新 `index.html` 或目录缓存。

---

## 19. 停止、重启与回滚

### 19.1 平滑重载配置

修改配置后优先使用 reload，不要直接 restart。

```bash
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl reload nginx
```

### 19.2 重启 Nginx

只有在 reload 无法满足要求，或升级二进制后才考虑 restart。

```bash
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl restart nginx
systemctl status nginx
```

### 19.3 停止 Nginx

```bash
systemctl stop nginx
systemctl status nginx
```

### 19.4 配置回滚

配置变更前应先备份：

```bash
cp /data/module/nginx/conf/conf.d/vue-app.conf /data/module/nginx/conf/conf.d/vue-app.conf.bak.$(date +%F-%H%M%S)
```

回滚示例：

```bash
cp /data/module/nginx/conf/conf.d/vue-app.conf.bak.2026-06-16-120000 /data/module/nginx/conf/conf.d/vue-app.conf
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl reload nginx
```

### 19.5 软件版本回滚

如果后续升级了 Nginx，可通过稳定软链接回滚到旧版本。

```bash
ln -sfn /data/module/nginx1.31.1 /data/module/nginx
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
systemctl restart nginx
```

注意：版本回滚前必须确认新旧版本配置指令兼容。

---

## 20. 日志切割

### 20.1 写入 logrotate 配置

```bash
cat > /etc/logrotate.d/nginx << 'EOF'
/data/module/nginx/logs/*.log {
    daily
    rotate 30
    missingok
    notifempty
    compress
    delaycompress
    dateext
    create 0640 nginx nginx
    sharedscripts
    postrotate
        if [ -f /data/module/nginx/run/nginx.pid ]; then
            /bin/kill -USR1 $(cat /data/module/nginx/run/nginx.pid)
        fi
    endscript
}
EOF
```

### 20.2 测试 logrotate 配置

```bash
logrotate -d /etc/logrotate.d/nginx
```

如需强制执行一次：

```bash
logrotate -f /etc/logrotate.d/nginx
```

验证：

```bash
ls -lh /data/module/nginx/logs
```

---

## 21. 常见问题

### 21.1 访问 Vue 子路由刷新后 404

现象：

```text
访问 /user/list 正常，刷新后 404
```

原因：Vue Router history 模式下，服务器没有真实的 `/user/list` 文件。

处理：确认站点配置包含：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

### 21.2 页面更新后用户仍看到旧版本

原因通常是 `index.html` 被浏览器、代理或 CDN 缓存。

处理：

- 确认 `index.html` 配置了 no-cache；
- 如果前置 CDN，需要刷新 CDN 缓存；
- 确认前端构建产物 JS/CSS 文件名带 hash；
- 避免将 `index.html` 设置为长期缓存。

### 21.3 API 请求 502 Bad Gateway

常见原因：

- 后端服务未启动；
- `upstream` 地址或端口配置错误；
- 防火墙阻断 Nginx 到后端连接；
- 后端只监听了错误的 IP；
- SELinux 阻止 Nginx 发起网络连接。

排查命令：

```bash
systemctl status nginx
ss -lntup | grep 8080
curl -I http://127.0.0.1:8080/health
tail -n 100 /data/module/nginx/logs/vue-app-error.log
```

### 21.4 API 请求 504 Gateway Timeout

常见原因：

- 后端接口处理时间超过 `proxy_read_timeout`；
- 后端线程池、数据库连接池或依赖服务阻塞；
- Nginx 到后端网络异常。

处理建议：

- 优先排查后端接口耗时；
- 不要盲目把 Nginx 超时时间调得过大；
- 对确实需要长时间执行的接口单独配置超时时间；
- 大任务建议改为异步任务，不建议通过 HTTP 长时间阻塞。

### 21.5 启动失败提示 bind failed

现象：

```text
bind() to 0.0.0.0:8443 failed (98: Address already in use)
```

排查：

```bash
ss -lntup | grep -E ':8088|:8443'
```

处理：确认占用进程用途。如果是上一版 Nginx 没退完，等 `systemctl stop nginx` 完全退出后再启动；如果是其他进程，必须先确认用途，不得直接 kill 未知生产进程。如果服务器之前用过 80/443，迁移后需要确保旧的 Nginx 实例已经停止并禁用开机自启。

### 21.6 worker_connections are not enough

现象：错误日志出现：

```text
worker_connections are not enough
```

处理：

- 提高 `worker_connections`；
- 提高 `worker_rlimit_nofile`；
- 提高 systemd `LimitNOFILE`；
- 检查是否存在大量异常长连接；
- 检查后端慢响应是否导致连接堆积。

### 21.7 上传文件失败或 413 Request Entity Too Large

原因：请求体超过 `client_max_body_size`。

处理：根据业务需要调整：

```nginx
client_max_body_size 100m;
```

不建议无上限放大，应结合业务上传大小、网关、后端限制统一规划。

### 21.8 HTTPS 证书不生效

排查：

```bash
/data/module/nginx/sbin/nginx -t -c /data/module/nginx/conf/nginx.conf
openssl x509 -in /data/module/nginx/conf/certs/example.com.crt -noout -subject -issuer -dates
openssl s_client -connect 127.0.0.1:8443 -servername example.com </dev/null
```

常见原因：

- 证书路径配置错误；
- 私钥与证书不匹配；
- 证书链不完整；
- 修改证书后未 reload；
- 客户端访问的域名与证书 CN/SAN 不匹配。

### 21.9 nginx_status 被外部访问

处理：确保状态页限制来源：

```nginx
location /nginx_status {
    stub_status;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

如监控服务器需要访问，只添加明确的监控服务器 IP，不要开放 `0.0.0.0/0`。

### 21.10 用 IP 访问 HTTPS 时浏览器报证书告警

现象：

```text
您的连接不是私密连接
NET::ERR_CERT_COMMON_NAME_INVALID
（或 Firefox: SSL_ERROR_BAD_CERT_DOMAIN）
```

原因：HTTPS 证书的 CN/SAN 字段绑定的是域名（如 `example.com`），不包含 IP。浏览器用 `https://203.0.113.10:8443/` 访问时，**校验"访问地址"和"证书签发对象"不一致**，于是报错。这是浏览器层面的安全机制，与 Nginx 配置无关。

处理建议（按推荐顺序）：

1. **优先用域名访问 HTTPS**：把服务器 IP 解析到一个域名（哪怕是 hosts 临时映射），通过域名访问 `https://example.com:8443/`，告警自动消失；
2. **临时用 IP 访问时走 HTTP 8088**：HTTP 不涉及证书校验，POC 内网验证、curl 调试都不受影响；
3. **可接受手动忽略**：浏览器点"高级 → 继续访问"，或 curl 加 `-k`，对内部测试可接受；
4. **不推荐**：为了消告警去签一张包含 IP SAN 的自签名证书，单体 POC 场景维护成本高于收益。

排查命令：

```bash
openssl x509 -in /data/module/nginx/conf/certs/example.com.crt \
  -noout -subject -ext subjectAltName
openssl s_client -connect 127.0.0.1:8443 -servername example.com </dev/null \
  2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```

输出中 `Subject Alternative Name` 字段会列出证书绑定的域名/IP 范围。
