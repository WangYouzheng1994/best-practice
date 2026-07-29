# Nexus Repository 3.94.1 单体生产级 Docker 配置部署手册

> **Author:** 王有政<br>
> **Encoding:** UTF-8<br>
> **日期:** 2026-07-29<br>
> **适用系统:** CentOS 7.9 x86_64 存量服务器<br>
> **Nexus 版本:** 3.94.1（产品构建 3.94.1-06）<br>
> **Nexus 镜像:** `sonatype/nexus3:3.94.1-ubi`<br>
> **PostgreSQL 镜像:** `postgres:17.10-bookworm`

> CentOS 7.9、Docker 26.1.4 均已退出厂商支持范围。本手册只用于暂时不能迁移的存量服务器。正式接入业务前必须通过第 4 节镜像运行门禁、完整验收和一次恢复演练；新服务器应部署在受支持的 EL9 系统上。

## 目录

1. [部署基线](#1-部署基线)
2. [资源、路径与端口规划](#2-资源路径与端口规划)
3. [部署前检查](#3-部署前检查)
4. [固定镜像并执行兼容性门禁](#4-固定镜像并执行兼容性门禁)
5. [创建目录和密钥](#5-创建目录和密钥)
6. [初始化 PostgreSQL](#6-初始化-postgresql)
7. [配置 Nexus 数据源](#7-配置-nexus-数据源)
8. [编写 Docker Compose 配置](#8-编写-docker-compose-配置)
9. [启动服务](#9-启动服务)
10. [配置 Nginx HTTPS](#10-配置-nginx-https)
11. [初始化 Nexus](#11-初始化-nexus)
12. [配置 Maven 客户端](#12-配置-maven-客户端)
13. [备份](#13-备份)
14. [恢复](#14-恢复)
15. [升级与回滚](#15-升级与回滚)
16. [日常运维与验收](#16-日常运维与验收)
17. [官方参考](#17-官方参考)

---

## 1. 部署基线

本手册采用以下固定结构：

- Nexus 与 PostgreSQL 由同一个 Docker Compose 项目管理；
- Nexus 使用外置 PostgreSQL，不允许以嵌入式 H2 作为本部署的生产数据库；
- PostgreSQL 只在 Compose 网络内提供服务，不发布宿主机 `5432` 端口；
- Nexus 只发布到宿主机回环地址 `127.0.0.1:18082`；
- 宿主机 Nginx 监听 `18080/18443`，统一提供 HTTPS 入口；
- Blob、数据库、配置和密钥全部持久化到 `/data`；
- 关闭匿名访问，发布账号不授予删除权限；
- Maven Release 仓库禁止覆盖发布，Snapshot 仓库允许重新发布；
- 数据库、Blob 和配置必须按同一备份时间点恢复。

该部署是单机结构，宿主机、Docker、Nexus、PostgreSQL 或本地磁盘故障都会造成服务中断。关键生产环境必须将备份复制到异机或对象存储，并定期执行恢复演练。

---

## 2. 资源、路径与端口规划

### 2.1 资源基线

以下配置适用于 Maven 制品为主的中小型研发团队，正式容量以压测和制品增长速度为准。

| 项目 | 基线 |
| --- | --- |
| CPU | 8 核 |
| 内存 | 16 GB |
| `/data` 可用空间 | 初始不少于 200 GB，使用率达到 80% 告警 |
| 文件系统 | 本地 SSD；XFS 必须为 `ftype=1` |
| Nexus 容器 | 4 CPU、8 GB 内存上限 |
| PostgreSQL 容器 | 2 CPU、4 GB 内存上限 |
| 时间同步 | chrony 正常，时区 `Asia/Shanghai` |

磁盘可用空间必须始终大于 4 GB。生产告警应同时覆盖 `/data` 使用率、inode、Docker 容器重启次数、Nexus 状态接口和备份结果。

### 2.2 路径规划

| 用途 | 路径 |
| --- | --- |
| Compose 与运维脚本 | `/data/module/nexus/compose` |
| Nexus 持久化数据 | `/data/module/nexus/data` |
| PostgreSQL 数据 | `/data/module/nexus/postgresql` |
| PostgreSQL 初始化脚本 | `/data/module/nexus/postgresql-init` |
| 本机密钥文件 | `/data/module/nexus/secrets` |
| 本机备份 | `/data/backup/nexus` |
| Nginx 站点配置 | `/data/module/nginx/conf/conf.d/nexus.conf` |
| TLS 证书 | `/data/module/nginx/conf/certs/nexus.example.com.crt` |
| TLS 私钥 | `/data/module/nginx/conf/certs/nexus.example.com.key` |

### 2.3 端口规划

| 端口 | 监听地址 | 用途 | 暴露策略 |
| --- | --- | --- | --- |
| `18080/tcp` | 宿主机业务地址 | Nginx HTTP 跳转入口 | 仅向受信研发网段开放 |
| `18443/tcp` | 宿主机业务地址 | Nginx HTTPS 业务入口 | 仅向受信研发网段开放 |
| `18082/tcp` | `127.0.0.1` | Nexus 宿主机回环入口 | 不对外开放 |
| `5432/tcp` | 仅 Compose 网络 | PostgreSQL | 不发布宿主机端口 |

文中的 `nexus.example.com`、证书路径和受信网段均为现场替换项。生产域名必须提前完成 DNS 解析并申请受信任证书。

---

## 3. 部署前检查

先完成以下仓库内手册：

- [CentOS 7.9 生产环境初始化手册](./CentOS7.9生产环境初始化手册.md)；
- [Docker 26.1.4 生产级安装配置手册](./Docker生产级安装配置运维手册.md)；
- [Nginx 1.31 单体生产级配置部署手册](./Nginx1.31单体生产级配置部署手册.md)。

执行检查：

```bash
cat /etc/centos-release
uname -m
uname -r

docker version
docker compose version
docker info | grep -E 'Docker Root Dir|Storage Driver|Backing Filesystem|Supports d_type'
systemctl is-active docker
systemctl is-active nginx

findmnt /data
df -hT /data
df -ih /data
free -h
nproc
timedatectl

ip -4 route
ss -lntp | grep -E ':(18080|18082|18443)[[:space:]]' || true
```

如果 `/data` 是 XFS，必须显示 `ftype=1`：

```bash
xfs_info /data | grep 'ftype=1'
```

确认以下条件后才能继续：

- `/data` 是持久化数据盘且备份目标可用；
- Docker `data-root` 为 `/data/docker`；
- Docker 地址池与企业内网、VPN、VPC 和专线网段不冲突；
- `18082` 没有被其他进程占用；
- `18080/18443` 如已由 Nginx 监听，可以继续增加基于域名的站点配置；
- 服务器能够访问 Maven Central，或已配置企业允许的上游代理；
- 已确定 Nexus Community Edition 或 Pro 的使用范围并符合对应许可条款。

---

## 4. 固定镜像并执行兼容性门禁

### 4.1 拉取固定版本

```bash
docker pull sonatype/nexus3:3.94.1-ubi
docker pull postgres:17.10-bookworm
```

不允许使用 `latest`。本手册固定使用 Nexus 官方 UBI + Java 21 变体；该选择不改变 CentOS 7.9 已停止支持的事实。

### 4.2 执行容器运行门禁

```bash
docker run --rm --platform linux/amd64 \
  --entrypoint /bin/sh \
  sonatype/nexus3:3.94.1-ubi \
  -c 'id && java -version'

docker run --rm --platform linux/amd64 \
  --entrypoint /bin/sh \
  postgres:17.10-bookworm \
  -c 'id postgres && postgres --version'
```

Nexus 当前用户必须为 UID/GID `200:200`，PostgreSQL 的 `postgres` 用户必须为 UID/GID `999:999`：

```bash
docker run --rm --entrypoint /bin/sh \
  sonatype/nexus3:3.94.1-ubi -c 'id -u; id -g'

docker run --rm --entrypoint /bin/sh \
  postgres:17.10-bookworm -c 'id -u postgres; id -g postgres'
```

任何容器无法启动、Java 21 无法运行、出现 `unsupported kernel`、非法指令或 UID/GID 不一致时，立即停止部署。不得在未验证数据库和升级路径的情况下直接改用旧版 Nexus。

### 4.3 记录实际镜像摘要

```bash
mkdir -p /data/module/nexus/compose

{
  date -Is
  docker image inspect \
    --format '{{range .RepoDigests}}{{println .}}{{end}}' \
    sonatype/nexus3:3.94.1-ubi
  docker image inspect \
    --format '{{range .RepoDigests}}{{println .}}{{end}}' \
    postgres:17.10-bookworm
} | tee /data/module/nexus/compose/image-digests.txt
```

镜像摘要必须随部署配置和备份保留。后续同一标签摘要发生变化时，按升级流程重新验证，不得静默替换生产镜像。

---

## 5. 创建目录和密钥

### 5.1 创建目录

```bash
install -d -m 0755 -o root -g root /data/module/nexus
install -d -m 0750 -o root -g root /data/module/nexus/compose
install -d -m 0750 -o 200  -g 200  /data/module/nexus/data
install -d -m 0700 -o 999  -g 999  /data/module/nexus/postgresql
install -d -m 0750 -o root -g root /data/module/nexus/postgresql-init
install -d -m 0700 -o root -g root /data/module/nexus/secrets
install -d -m 0700 -o root -g root /data/backup/nexus

install -d -m 0750 -o 200 -g 200 \
  /data/module/nexus/data/etc/fabric
```

### 5.2 生成数据库密码

```bash
umask 077

openssl rand -hex 32 \
  > /data/module/nexus/secrets/postgres-admin-password.txt

openssl rand -hex 32 \
  > /data/module/nexus/secrets/nexus-db-password.txt

chown 999:999 /data/module/nexus/secrets/*.txt
chmod 0400 /data/module/nexus/secrets/*.txt
```

要求：

- 两个密码必须不同；
- 密码不得写入 Compose、Git、Wiki、工单或聊天记录；
- 密钥文件只允许宿主机 root 和 PostgreSQL 容器内 UID `999` 读取；
- 密钥备份必须加密并纳入访问审计。

---

## 6. 初始化 PostgreSQL

写入初始化脚本：

```bash
cat > /data/module/nexus/postgresql-init/01-init-nexus.sh << 'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

NEXUS_DB_PASSWORD="$(tr -d '\r\n' < /run/secrets/nexus_db_password)"

psql --set ON_ERROR_STOP=1 \
  --username "${POSTGRES_USER}" \
  --dbname postgres \
  --set=nexus_password="${NEXUS_DB_PASSWORD}" <<'EOSQL'
CREATE ROLE nexus
  LOGIN
  PASSWORD :'nexus_password'
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE
  NOREPLICATION
  NOBYPASSRLS;

CREATE DATABASE nexus
  OWNER nexus
  ENCODING 'UTF8'
  TEMPLATE template0;
EOSQL

psql --set ON_ERROR_STOP=1 \
  --username "${POSTGRES_USER}" \
  --dbname nexus <<'EOSQL'
SET ROLE nexus;
CREATE SCHEMA nexus AUTHORIZATION nexus;
CREATE EXTENSION pg_trgm SCHEMA nexus;
RESET ROLE;

REVOKE CREATE ON SCHEMA public FROM PUBLIC;
EOSQL

unset NEXUS_DB_PASSWORD
EOF

chown root:root /data/module/nexus/postgresql-init/01-init-nexus.sh
chmod 0555 /data/module/nexus/postgresql-init/01-init-nexus.sh
```

初始化结果必须满足：

- `nexus` 是数据库所有者；
- `nexus` 不是超级用户，也没有创建数据库和角色的权限；
- `nexus` schema 及 `pg_trgm` 扩展的所有者均为 `nexus`；
- Nexus 通过 `currentSchema=nexus` 使用独立 schema。

该脚本只在 PostgreSQL 数据目录为空时执行。已经初始化的实例修改脚本不会自动修改数据库账号或结构。

---

## 7. 配置 Nexus 数据源

使用 `nexus-store.properties` 提供数据库配置，避免将数据库密码暴露在容器环境变量和 `docker inspect` 输出中。

```bash
NEXUS_DB_PASSWORD="$(tr -d '\r\n' \
  < /data/module/nexus/secrets/nexus-db-password.txt)"

if ! [[ "${NEXUS_DB_PASSWORD}" =~ ^[0-9a-f]{64}$ ]]; then
  echo 'nexus 数据库密码格式不正确' >&2
  unset NEXUS_DB_PASSWORD
  exit 1
fi

cat > /data/module/nexus/data/etc/fabric/nexus-store.properties << EOF
username=nexus
password=${NEXUS_DB_PASSWORD}
jdbcUrl=jdbc\:postgresql\://postgresql\:5432/nexus?currentSchema=nexus
EOF

unset NEXUS_DB_PASSWORD

chown 200:200 \
  /data/module/nexus/data/etc/fabric/nexus-store.properties
chmod 0600 \
  /data/module/nexus/data/etc/fabric/nexus-store.properties
```

不得同时配置 `NEXUS_DATASTORE_NEXUS_*` 环境变量、JVM 数据源参数和 `nexus-store.properties`。本手册只使用属性文件这一种连接方式。

---

## 8. 编写 Docker Compose 配置

写入 `/data/module/nexus/compose/compose.yml`：

```bash
cat > /data/module/nexus/compose/compose.yml << 'EOF'
name: nexus

services:
  postgresql:
    image: postgres:17.10-bookworm
    platform: linux/amd64
    container_name: nexus-postgresql
    hostname: postgresql
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
      POSTGRES_USER: postgres
      POSTGRES_DB: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_admin_password
      POSTGRES_INITDB_ARGS: >-
        --auth-host=scram-sha-256
        --encoding=UTF8
        --locale=C.UTF-8
      PGDATA: /var/lib/postgresql/data/pgdata
    command:
      - postgres
      - -c
      - max_connections=200
      - -c
      - shared_buffers=1GB
      - -c
      - effective_cache_size=3GB
      - -c
      - work_mem=8MB
      - -c
      - maintenance_work_mem=256MB
      - -c
      - checkpoint_completion_target=0.9
      - -c
      - min_wal_size=1GB
      - -c
      - max_wal_size=4GB
      - -c
      - wal_compression=on
      - -c
      - log_min_duration_statement=1000
      - -c
      - log_checkpoints=on
      - -c
      - log_connections=on
      - -c
      - log_disconnections=on
      - -c
      - log_lock_waits=on
      - -c
      - timezone=Asia/Shanghai
      - -c
      - log_timezone=Asia/Shanghai
    volumes:
      - /data/module/nexus/postgresql:/var/lib/postgresql/data:Z
      - /data/module/nexus/postgresql-init/01-init-nexus.sh:/docker-entrypoint-initdb.d/01-init-nexus.sh:ro,Z
      - /data/module/nexus/secrets/postgres-admin-password.txt:/run/secrets/postgres_admin_password:ro,Z
      - /data/module/nexus/secrets/nexus-db-password.txt:/run/secrets/nexus_db_password:ro,Z
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h 127.0.0.1 -U postgres -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 20s
    shm_size: 1gb
    mem_limit: 4g
    mem_reservation: 2g
    cpus: 2.0
    pids_limit: 1024
    stop_grace_period: 60s
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    networks:
      - nexus-net

  nexus:
    image: sonatype/nexus3:3.94.1-ubi
    platform: linux/amd64
    container_name: nexus
    hostname: nexus
    restart: unless-stopped
    depends_on:
      postgresql:
        condition: service_healthy
    environment:
      TZ: Asia/Shanghai
      INSTALL4J_ADD_VM_PARAMS: >-
        -Xms2703m
        -Xmx2703m
        -XX:MaxDirectMemorySize=2703m
        -Djava.util.prefs.userRoot=/nexus-data/javaprefs
        -Dfile.encoding=UTF-8
    ports:
      - "127.0.0.1:18082:8081"
    volumes:
      - /data/module/nexus/data:/nexus-data:Z
    mem_limit: 8g
    mem_reservation: 6g
    cpus: 4.0
    pids_limit: 4096
    stop_grace_period: 120s
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    security_opt:
      - no-new-privileges:true
    networks:
      - nexus-net

networks:
  nexus-net:
    name: nexus-net
    driver: bridge
EOF

chown root:root /data/module/nexus/compose/compose.yml
chmod 0640 /data/module/nexus/compose/compose.yml
```

本配置不为 PostgreSQL 声明 `ports`，Nexus 与 Maven Central 的出站访问通过普通 bridge 网络完成。

校验 Compose 最终配置：

```bash
cd /data/module/nexus/compose
docker compose config --quiet
docker compose config --images
```

输出只能包含以下两个固定镜像：

```text
postgres:17.10-bookworm
sonatype/nexus3:3.94.1-ubi
```

---

## 9. 启动服务

### 9.1 启动 PostgreSQL

```bash
cd /data/module/nexus/compose
docker compose up -d postgresql
docker compose ps
docker compose logs --tail=200 postgresql
```

等待 PostgreSQL 变为 `healthy`，然后验收数据库：

```bash
docker compose exec -T postgresql \
  psql -U postgres -d postgres -c \
  "SELECT datname, pg_get_userbyid(datdba) AS owner FROM pg_database WHERE datname='nexus';"

docker compose exec -T postgresql \
  psql -U postgres -d nexus -c \
  "SELECT nspname, nspowner::regrole FROM pg_namespace WHERE nspname='nexus';"

docker compose exec -T postgresql \
  psql -U postgres -d nexus -c \
  "SELECT extname, extowner::regrole, extnamespace::regnamespace FROM pg_extension WHERE extname='pg_trgm';"

docker compose exec -T postgresql \
  psql -U postgres -d postgres -c '\du nexus'
```

数据库、schema、扩展所有者必须为 `nexus`，且该角色不得显示 Superuser、Create role 或 Create DB。

### 9.2 启动 Nexus

```bash
cd /data/module/nexus/compose
docker compose up -d nexus
docker compose ps
docker compose logs --tail=300 nexus
```

首次启动通常需要数分钟。等待状态接口返回成功：

```bash
ready=0
for i in $(seq 1 60); do
  if curl -fsS \
    http://127.0.0.1:18082/service/rest/v1/status/writable; then
    echo
    echo 'Nexus 已就绪'
    ready=1
    break
  fi
  sleep 10
done

if [ "${ready}" -ne 1 ]; then
  echo 'Nexus 未在 10 分钟内进入可写状态' >&2
  exit 1
fi

curl -fsS http://127.0.0.1:18082/service/rest/v1/status
curl -fsS http://127.0.0.1:18082/service/rest/v1/status/writable
```

确认端口没有错误暴露：

```bash
docker port nexus
docker port nexus-postgresql
ss -lntp | grep ':18082[[:space:]]'

docker inspect --format \
  '{{.Name}} memory={{.HostConfig.Memory}} nano_cpus={{.HostConfig.NanoCpus}} pids={{.HostConfig.PidsLimit}}' \
  nexus nexus-postgresql
```

要求：

- Nexus 只显示 `127.0.0.1:18082`；
- PostgreSQL 不显示任何宿主机端口；
- Nexus/PostgreSQL 的内存、CPU 和 PID 限制与 Compose 一致；
- 日志中不得出现 H2 JDBC URL、数据库鉴权失败或 schema 初始化失败。

---

## 10. 配置 Nginx HTTPS

### 10.1 安装证书

将生产证书和私钥放入约定目录，替换示例域名：

```bash
chown root:root \
  /data/module/nginx/conf/certs/nexus.example.com.crt \
  /data/module/nginx/conf/certs/nexus.example.com.key

chmod 0644 /data/module/nginx/conf/certs/nexus.example.com.crt
chmod 0600 /data/module/nginx/conf/certs/nexus.example.com.key
```

不得使用自签名证书作为团队生产入口。使用企业内部 CA 时，必须将根证书统一下发到开发机、构建机和 JDK 信任库。

### 10.2 写入站点配置

以下配置中的 `nexus.example.com` 必须全部替换为正式域名：

```bash
cat > /data/module/nginx/conf/conf.d/nexus.conf << 'EOF'
upstream nexus_repository {
    server 127.0.0.1:18082;
    keepalive 32;
}

server {
    listen 18080;
    server_name nexus.example.com;

    return 301 https://nexus.example.com:18443$request_uri;
}

server {
    listen 18443 ssl;
    http2 on;
    server_name nexus.example.com;

    ssl_certificate     /data/module/nginx/conf/certs/nexus.example.com.crt;
    ssl_certificate_key /data/module/nginx/conf/certs/nexus.example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:NEXUS_SSL:10m;
    ssl_session_timeout 1d;

    client_max_body_size 5g;
    client_body_timeout 600s;

    access_log /data/module/nginx/logs/nexus.access.log main;
    error_log  /data/module/nginx/logs/nexus.error.log warn;

    location / {
        proxy_pass http://nexus_repository;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 18443;

        proxy_connect_timeout 10s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
        proxy_request_buffering off;
        proxy_buffering off;
        proxy_redirect off;
    }
}
EOF

chown root:root /data/module/nginx/conf/conf.d/nexus.conf
chmod 0644 /data/module/nginx/conf/conf.d/nexus.conf
```

如果 SELinux 为 Enforcing，允许 Nginx 连接回环上游：

```bash
if [ "$(getenforce)" = 'Enforcing' ]; then
  setsebool -P httpd_can_network_connect 1
fi
```

上述布尔值只处理 Nginx 连接上游的权限。SELinux 为 Enforcing 时，还必须依据企业 SELinux 策略为 `/data/module/nginx` 的配置、证书、日志和运行目录配置持久化 file context，并通过 `ausearch -m AVC -ts recent` 复核；存在未处理的 AVC 拒绝时停止上线，不得直接关闭 SELinux 规避。

校验并加载配置：

```bash
/data/module/nginx/sbin/nginx -t
systemctl reload nginx
systemctl status nginx --no-pager
```

### 10.3 配置网络边界

CentOS 7.9 初始化基线默认关闭本机 `firewalld`，不得为 Nexus 单独启用。由网络或运维人员在外层防火墙、安全组、ACL、VPN 或统一网络策略中执行：

- 只允许研发网、构建网和批准的代理出口访问 `18080/18443`；
- 不为 `18082` 或 `5432` 配置外部入站规则；
- 将来源网段、端口和用途登记到网络策略及资产台账；
- 需要公网接入时，由统一入口完成 WAF、TLS 和 NAT，Nexus 宿主机不得直接暴露公网。

如果现场原本启用了 `firewalld`，按《CentOS 7.9 生产环境初始化手册》的现场例外流程增加同等来源限制，不得直接使用无来源约束的 `--add-port`。

验证 HTTPS：

```bash
curl --fail --silent --show-error \
  --resolve nexus.example.com:18443:127.0.0.1 \
  https://nexus.example.com:18443/service/rest/v1/status/writable
```

---

## 11. 初始化 Nexus

### 11.1 完成首次登录

读取一次性管理员密码：

```bash
cd /data/module/nexus/compose
docker compose exec nexus cat /nexus-data/admin.password
```

访问：

```text
https://nexus.example.com:18443/
```

使用 `admin` 登录并立即执行：

1. 修改管理员密码并存入企业密码管理系统；
2. 关闭 Anonymous Access；
3. 创建实名运维管理员，授予 `nx-admin`；
4. 将内置 `admin` 作为受控的应急账号，不用于日常操作；
5. 在 Base URL capability 中设置 `https://nexus.example.com:18443/`；
6. 保持 HTTP Forwarded Headers capability 启用；
7. 确认 Data Store 显示 PostgreSQL JDBC URL，不得显示 H2。

### 11.2 创建 Blob Store

在 `Administration -> Repository -> Blob Stores` 创建：

| 名称 | 类型 | 用途 |
| --- | --- | --- |
| `maven-hosted` | File | 企业 Release 与 Snapshot 制品 |
| `maven-cache` | File | Maven Central 代理缓存与 Group 缓存 |

Blob Store 的 Soft Quota 按实际磁盘规划设置为容量上限的 80%。不得把 PostgreSQL 数据目录作为 Blob 路径。

### 11.3 创建 Maven 仓库

在任何客户端接入前完成下表配置。全新实例中如果已自动创建同名默认仓库，应按下表调整；Blob Store 无法直接调整时，在无业务数据的前提下按 `maven-public -> maven-central -> maven-snapshots -> maven-releases` 的顺序删除后重建。

| 仓库 | Recipe | Blob Store | 关键配置 |
| --- | --- | --- | --- |
| `maven-releases` | `maven2 (hosted)` | `maven-hosted` | Version policy=`Release`；Layout policy=`Strict`；Deployment policy=`Disable redeploy` |
| `maven-snapshots` | `maven2 (hosted)` | `maven-hosted` | Version policy=`Snapshot`；Layout policy=`Strict`；Deployment policy=`Allow` |
| `maven-central` | `maven2 (proxy)` | `maven-cache` | Remote storage=`https://repo.maven.apache.org/maven2/`；Version policy=`Release`；Layout policy=`Strict`；启用 Content Type Validation |
| `maven-public` | `maven2 (group)` | `maven-cache` | Version policy=`Mixed`；成员依次为 `maven-releases`、`maven-snapshots`、`maven-central` |

仓库地址：

```text
读取聚合仓库：https://nexus.example.com:18443/repository/maven-public/
Release 发布：https://nexus.example.com:18443/repository/maven-releases/
Snapshot 发布：https://nexus.example.com:18443/repository/maven-snapshots/
```

### 11.4 创建最小权限角色和账号

创建角色：

| Role ID | 角色权限 |
| --- | --- |
| `maven-reader` | `nx-repository-view-maven2-maven-public-browse`、`nx-repository-view-maven2-maven-public-read` |
| `maven-snapshot-publisher` | `maven-reader` 角色，加 `maven-snapshots` 的 `browse/read/add/edit` |
| `maven-release-publisher` | `maven-reader` 角色，加 `maven-releases` 的 `browse/read/add/edit` |

发布权限对应的完整 privilege 前缀为：

```text
nx-repository-view-maven2-maven-snapshots-*
nx-repository-view-maven2-maven-releases-*
```

只选择 `browse`、`read`、`add`、`edit`，不得选择 `delete`。Release 仓库的 `Disable redeploy` 是不可覆盖发布的第二道控制。

至少创建以下服务账号：

| 账号示例 | 角色 | 用途 |
| --- | --- | --- |
| `ci-reader` | `maven-reader` | CI/CD 下载依赖 |
| `ci-snapshot` | `maven-snapshot-publisher` | CI 发布 Snapshot |
| `ci-release` | `maven-release-publisher` | 受控 Release 流水线发布 |

服务账号密码只保存在 CI/CD Secret 管理系统中，不得写入项目 POM、`settings.xml` 模板或流水线仓库。开发人员使用各自的 `maven-reader` 账号，不共享 CI 服务账号。

### 11.5 配置清理任务

生产环境至少配置以下策略：

- `maven-snapshots`：清理超过 30 天未更新的 Snapshot；
- `maven-central`：清理超过 90 天未下载的代理缓存；
- `maven-releases`：不配置自动删除策略；
- 每日执行 Repository Cleanup；
- Cleanup 完成后每周执行 Blob Store Compact；
- 清理参数变更前先检查 Preview，不得直接将策略应用到 Release 仓库。

---

## 12. 配置 Maven 客户端

### 12.1 团队 `settings.xml` 模板

开发机和构建机统一使用以下模板，实际账号密码通过环境变量注入：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
  <mirrors>
    <mirror>
      <id>nexus-mirror</id>
      <name>Company Nexus</name>
      <url>https://nexus.example.com:18443/repository/maven-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>

  <servers>
    <server>
      <id>nexus-mirror</id>
      <username>${env.NEXUS_READ_USERNAME}</username>
      <password>${env.NEXUS_READ_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-releases</id>
      <username>${env.NEXUS_RELEASE_USERNAME}</username>
      <password>${env.NEXUS_RELEASE_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>${env.NEXUS_SNAPSHOT_USERNAME}</username>
      <password>${env.NEXUS_SNAPSHOT_PASSWORD}</password>
    </server>
  </servers>

  <profiles>
    <profile>
      <id>company-nexus</id>
      <repositories>
        <repository>
          <id>nexus-public</id>
          <url>https://nexus.example.com:18443/repository/maven-public/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>nexus-public-plugins</id>
          <url>https://nexus.example.com:18443/repository/maven-public/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>company-nexus</activeProfile>
  </activeProfiles>
</settings>
```

个人文件保存到 `~/.m2/settings.xml` 并执行：

```bash
chmod 0600 ~/.m2/settings.xml
```

禁止使用 `maven.wagon.http.ssl.insecure=true` 或关闭证书校验来绕过 TLS 问题。

### 12.2 项目发布配置

项目 POM 使用与 `settings.xml` 一致的 Server ID：

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>https://nexus.example.com:18443/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>https://nexus.example.com:18443/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

Release 只能由受控流水线执行，普通开发账号不得持有 Release 发布角色。

---

## 13. 备份

### 13.1 备份原则

每个可恢复备份集必须来自同一时间点，并同时包含：

- PostgreSQL `nexus` 数据库逻辑备份；
- PostgreSQL 全局角色备份；
- 完整 `/nexus-data`，其中必须包含 Blob、配置、密钥库和节点标识目录 `/nexus-data/keystores/node/`；
- Compose、初始化脚本、数据库连接属性和密钥文件；
- 镜像摘要和校验文件。

备份期间先优雅停止 Nexus，PostgreSQL 保持运行以执行逻辑备份。停止宽限期不得少于 120 秒。

当前脚本会先生成明文临时归档，再生成加密备份集。`/data/backup` 建议使用独立备份盘；可用空间至少应为当前 `/nexus-data` 实际占用量的 2.2 倍，避免备份过程占满 Nexus 数据盘。

### 13.2 配置备份加密公钥

备份集包含数据库角色密码哈希和 Nexus 数据库明文连接密码，必须使用企业备份公钥加密。私钥只保存在受控密码系统或灾备环境，不得存放在 Nexus 宿主机。

```bash
yum install -y gnupg2

gpg2 --import /root/nexus-backup-public.asc
gpg2 --list-keys --with-fingerprint

BACKUP_GPG_FINGERPRINT='替换为40位十六进制公钥指纹'
printf '%s\n' "${BACKUP_GPG_FINGERPRINT}" \
  > /data/module/nexus/secrets/backup-gpg-recipient.txt

chown root:root \
  /data/module/nexus/secrets/backup-gpg-recipient.txt
chmod 0400 \
  /data/module/nexus/secrets/backup-gpg-recipient.txt

gpg2 --batch --list-keys "${BACKUP_GPG_FINGERPRINT}"
unset BACKUP_GPG_FINGERPRINT
```

最后一条命令必须准确显示已审批的备份公钥和指纹。

### 13.3 安装备份脚本

```bash
cat > /data/module/nexus/compose/backup.sh << 'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

COMPOSE_DIR=/data/module/nexus/compose
BACKUP_ROOT=/data/backup/nexus
BACKUP_DIR="${BACKUP_ROOT}/$(date +%Y%m%d-%H%M%S)"
LOCK_FILE=/var/lock/nexus-backup.lock
RECIPIENT_FILE=/data/module/nexus/secrets/backup-gpg-recipient.txt
nexus_stopped=0

BACKUP_GPG_RECIPIENT="$(tr -d '[:space:]' < "${RECIPIENT_FILE}")"
if ! [[ "${BACKUP_GPG_RECIPIENT}" =~ ^[0-9A-Fa-f]{40}$ ]]; then
  echo '备份公钥指纹格式不正确' >&2
  exit 1
fi

gpg2 --batch --list-keys "${BACKUP_GPG_RECIPIENT}" >/dev/null 2>&1

exec 9>"${LOCK_FILE}"
flock -n 9 || {
  echo '已有 Nexus 备份任务运行中' >&2
  exit 1
}

cleanup() {
  rc=$?
  trap - EXIT
  rm -f -- \
    "${BACKUP_DIR}/nexus.dump" \
    "${BACKUP_DIR}/globals.sql" \
    "${BACKUP_DIR}/nexus-files.tar.gz" \
    "${BACKUP_DIR}/SHA256SUMS" \
    "${BACKUP_DIR}/nexus-backup.tar.gpg.tmp"
  if [ "${nexus_stopped}" -eq 1 ]; then
    cd "${COMPOSE_DIR}"
    docker compose up -d nexus || rc=1
  fi
  exit "${rc}"
}
trap cleanup EXIT

mkdir -p "${BACKUP_DIR}"
cd "${COMPOSE_DIR}"

docker compose stop -t 120 nexus
nexus_stopped=1

docker compose exec -T postgresql \
  pg_dump -U postgres -d nexus -Fc \
  > "${BACKUP_DIR}/nexus.dump"

docker compose exec -T postgresql \
  pg_dumpall -U postgres --globals-only \
  > "${BACKUP_DIR}/globals.sql"

tar --xattrs --acls --numeric-owner \
  -C /data/module/nexus \
  -czf "${BACKUP_DIR}/nexus-files.tar.gz" \
  data compose postgresql-init secrets

(
  cd "${BACKUP_DIR}"
  sha256sum nexus.dump globals.sql nexus-files.tar.gz > SHA256SUMS

  tar -cf - \
    nexus.dump globals.sql nexus-files.tar.gz SHA256SUMS \
    | gpg2 --batch --yes --trust-model always \
      --recipient "${BACKUP_GPG_RECIPIENT}" \
      --output nexus-backup.tar.gpg.tmp \
      --encrypt

  mv nexus-backup.tar.gpg.tmp nexus-backup.tar.gpg
  sha256sum nexus-backup.tar.gpg \
    > nexus-backup.tar.gpg.sha256

  rm -f -- nexus.dump globals.sql nexus-files.tar.gz SHA256SUMS
)

docker compose up -d nexus
nexus_stopped=0

ready=0
for i in $(seq 1 60); do
  if curl -fsS -o /dev/null \
    http://127.0.0.1:18082/service/rest/v1/status/writable; then
    ready=1
    break
  fi
  sleep 10
done

if [ "${ready}" -ne 1 ]; then
  echo '备份完成，但 Nexus 未在规定时间内恢复就绪' >&2
  exit 1
fi

trap - EXIT
echo "Nexus backup completed: ${BACKUP_DIR}"
EOF

chown root:root /data/module/nexus/compose/backup.sh
chmod 0700 /data/module/nexus/compose/backup.sh
```

先手工执行并确认结果：

```bash
/data/module/nexus/compose/backup.sh

LATEST_BACKUP="$(find /data/backup/nexus -mindepth 1 -maxdepth 1 \
  -type d -printf '%f\n' | sort | tail -n 1)"

cd "/data/backup/nexus/${LATEST_BACKUP}"
sha256sum -c nexus-backup.tar.gpg.sha256
ls -lh
```

每个备份目录最终只应保留 `nexus-backup.tar.gpg` 和 `nexus-backup.tar.gpg.sha256`；不得残留明文 dump、配置归档或密钥文件。

确认恢复演练成功后，再接入企业备份调度。示例每日 02:00 执行：

```bash
cat > /etc/cron.d/nexus-backup << 'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

0 2 * * * root /data/module/nexus/compose/backup.sh
EOF

chown root:root /etc/cron.d/nexus-backup
chmod 0644 /etc/cron.d/nexus-backup
systemctl reload crond
```

备份完成后必须将加密文件及其校验文件复制到异机或受控对象存储。本机 `/data/backup` 不能作为唯一备份。保留周期由企业备份制度统一管理，不在脚本内自动删除备份。

---

## 14. 恢复

以下流程用于在同版本镜像上恢复完整备份集。恢复前停止构建和发布任务，并预留一份现有数据目录所需的额外空间。

第 14 节所有命令必须在同一个 root Bash 会话中按顺序执行，保留 `BACKUP_DIR`、`RESTORE_WORK_DIR` 和 `RESTORE_TS` 的上下文；任一命令失败立即停止。

### 14.1 校验并解密备份

```bash
BACKUP_SET_DIR=/data/backup/nexus/20260729-020000
RESTORE_WORK_DIR="/data/backup/nexus/.restore-$(date +%Y%m%d-%H%M%S)"

test -d "${BACKUP_SET_DIR}"
cd "${BACKUP_SET_DIR}"
sha256sum -c nexus-backup.tar.gpg.sha256

install -d -m 0700 -o root -g root "${RESTORE_WORK_DIR}"

gpg2 --decrypt \
  "${BACKUP_SET_DIR}/nexus-backup.tar.gpg" \
  | tar -xf - -C "${RESTORE_WORK_DIR}"

BACKUP_DIR="${RESTORE_WORK_DIR}"
cd "${BACKUP_DIR}"
sha256sum -c SHA256SUMS
```

解密操作必须由获批的灾备私钥持有人执行。外层加密文件和内部三个文件必须全部校验成功；不得混用不同时间点的 `nexus.dump` 与 `nexus-files.tar.gz`。解密目录包含敏感明文，恢复验收完成后必须按企业安全流程立即清理。

### 14.2 停止服务并保留故障现场

```bash
cd /data/module/nexus/compose
docker compose down --timeout 120

RESTORE_TS="$(date +%Y%m%d-%H%M%S)"
cd /data/module
mv nexus "nexus.failed.${RESTORE_TS}"

install -d -m 0755 -o root -g root /data/module/nexus
tar --xattrs --acls --numeric-owner \
  -xzf "${BACKUP_DIR}/nexus-files.tar.gz" \
  -C /data/module/nexus

install -d -m 0700 -o 999 -g 999 \
  /data/module/nexus/postgresql
```

### 14.3 初始化空数据库并恢复

```bash
cd /data/module/nexus/compose
docker compose config --quiet
docker compose up -d postgresql

postgres_ready=0
for i in $(seq 1 30); do
  if docker compose exec -T postgresql \
    pg_isready -h 127.0.0.1 -U postgres -d postgres; then
    postgres_ready=1
    break
  fi
  sleep 5
done

if [ "${postgres_ready}" -ne 1 ]; then
  echo 'PostgreSQL 未在规定时间内完成初始化' >&2
  exit 1
fi

docker compose exec -T postgresql \
  psql -U postgres -d postgres -c \
  "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='nexus' AND pid <> pg_backend_pid();"

docker compose exec -T postgresql \
  dropdb -U postgres --if-exists nexus

docker compose exec -T postgresql \
  createdb -U postgres -O nexus -E UTF8 -T template0 nexus

docker compose exec -T postgresql \
  pg_restore -U postgres -d nexus \
  --exit-on-error --no-owner --role=nexus \
  < "${BACKUP_DIR}/nexus.dump"
```

`globals.sql` 用于角色和权限核查。标准恢复由已备份的密钥及初始化脚本重建 `postgres/nexus` 角色，不直接导入 `globals.sql`；存在额外数据库角色时，由 DBA 审核后选择性恢复。

验证扩展和所有者：

```bash
docker compose exec -T postgresql \
  psql -U postgres -d nexus -c \
  "SELECT extname, extowner::regrole, extnamespace::regnamespace FROM pg_extension WHERE extname='pg_trgm';"
```

### 14.4 启动并验收

```bash
docker compose up -d nexus

ready=0
for i in $(seq 1 60); do
  if curl -fsS \
    http://127.0.0.1:18082/service/rest/v1/status/writable; then
    echo
    ready=1
    break
  fi
  sleep 10
done

if [ "${ready}" -ne 1 ]; then
  echo '恢复后的 Nexus 未在 10 分钟内进入可写状态' >&2
  exit 1
fi

docker compose ps
docker compose logs --tail=300 nexus
```

确认仓库、角色、用户、Blob、组件数量和 Maven 下载/发布均正常后，保留 `nexus.failed.<时间>` 至变更观察期结束，再按企业变更流程清理。

如果恢复失败，停止新实例，将 `/data/module/nexus` 改名并把 `nexus.failed.<时间>` 原样改回 `/data/module/nexus`，然后使用原 Compose 和原镜像重新启动。

---

## 15. 升级与回滚

### 15.1 升级前要求

每次升级必须完成：

1. 阅读目标版本 Release Notes、升级路径和 PostgreSQL 要求；
2. 运行第 13 节备份并验证校验和；
3. 在隔离环境用备份副本完成升级和 Maven 回归；
4. 记录新镜像 tag 与 digest；
5. 预留至少 120 秒停止时间和足够数据库迁移时间；
6. Nexus 与 PostgreSQL 大版本不得在同一个变更窗口同时升级。

修改为已验证的固定 Nexus 镜像后执行：

```bash
cd /data/module/nexus/compose
docker compose config --quiet
docker compose pull nexus
docker compose up -d nexus
docker compose logs --tail=300 nexus
curl -fsS http://127.0.0.1:18082/service/rest/v1/status
curl -fsS http://127.0.0.1:18082/service/rest/v1/status/writable
```

### 15.2 回滚规则

- Nexus 启动后可能升级 PostgreSQL schema，不能只把镜像 tag 改回旧版本；
- 需要回滚时，必须同时恢复旧 Nexus 镜像、升级前数据库、升级前 `/nexus-data`、配置和密钥；
- PostgreSQL 跨大版本不能直接复用旧数据目录，必须使用官方 `pg_upgrade` 或逻辑备份恢复流程；
- 不支持把已运行的新版本 Nexus 直接降级启动到旧数据上。

完整回滚按第 14 节使用升级前同一备份集恢复。

---

## 16. 日常运维与验收

### 16.1 常用命令

```bash
cd /data/module/nexus/compose

docker compose ps
docker compose logs --tail=200 nexus
docker compose logs --tail=200 postgresql

docker compose restart nexus
docker compose stop -t 120 nexus
docker compose up -d nexus

curl -fsS http://127.0.0.1:18082/service/rest/v1/status

df -h /data
df -ih /data
du -sh /data/module/nexus/data/blobs/*
du -sh /data/module/nexus/postgresql
```

Nexus Status API 不检查外部 PostgreSQL 可达性和磁盘读写，必须同时监控 PostgreSQL、`/data` 容量和实际 Maven 读写探针。

不要使用 `docker compose down -v`。本手册使用 bind mount，但该命令会形成错误运维习惯，并可能删除同项目后续增加的命名卷。

### 16.2 最终验收清单

- [ ] 镜像运行门禁通过，实际 digest 已记录；
- [ ] Nexus 版本为 `3.94.1-06`，使用 `3.94.1-ubi` 镜像；
- [ ] PostgreSQL 版本为 `17.10`，容器状态为 `healthy`；
- [ ] Nexus 数据源为 PostgreSQL，不是 H2；
- [ ] `/service/rest/v1/status/writable` 返回成功；
- [ ] `nexus` 是数据库、schema 和 `pg_trgm` 的所有者，但不是超级用户；
- [ ] PostgreSQL 没有宿主机端口，Nexus 只绑定 `127.0.0.1:18082`；
- [ ] HTTPS 证书链、域名和有效期正确，HTTP 能跳转到 HTTPS；
- [ ] 外层网络策略只向受信网段开放 `18080/18443`；
- [ ] 匿名访问已关闭；
- [ ] Reader、Snapshot Publisher、Release Publisher 权限验证通过；
- [ ] 发布账号没有 delete 权限；
- [ ] Release 不能覆盖发布，Snapshot 可以按策略更新；
- [ ] 通过 `maven-public` 成功下载 `junit:junit:4.13.2`；
- [ ] CI 分别成功发布一个 Snapshot 和一个 Release 测试制品；
- [ ] 宿主机重启后两个容器自动拉起，Maven 下载和发布回归通过；
- [ ] 清理策略只应用于 Snapshot 和代理缓存；
- [ ] 备份已复制到异机，校验和验证通过；
- [ ] 备份归档包含 `/nexus-data/keystores/node/`；
- [ ] 已使用备份副本完成一次隔离恢复演练；
- [ ] 已配置磁盘、状态接口、容器重启和备份失败告警。

---

## 17. 官方参考

- [Nexus Repository 系统要求](https://help.sonatype.com/en/sonatype-nexus-repository-system-requirements.html)
- [Nexus Repository 功能矩阵](https://help.sonatype.com/en/nexus-repository-feature-matrix.html)
- [使用 PostgreSQL 安装 Nexus Repository](https://help.sonatype.com/en/install-nexus-repository-with-a-postgresql-database.html)
- [Nexus Repository Data Store 配置](https://help.sonatype.com/en/data-store.html)
- [PostgreSQL pg_trgm 安装要求](https://help.sonatype.com/en/installing-the-trigram-module.html)
- [Nexus Repository 官方 Docker 镜像仓库](https://github.com/sonatype/docker-nexus3)
- [Nexus Repository 3.94.1-06 发布记录](https://github.com/sonatype/nexus-public/releases/tag/release-3.94.1-06)
- [Nexus Repository 反向代理配置](https://help.sonatype.com/en/run-behind-a-reverse-proxy.html)
- [Nexus Repository Status API](https://help.sonatype.com/en/status-api.html)
- [Nexus Repository 备份准备](https://help.sonatype.com/en/prepare-a-backup.html)
- [Nexus Repository 升级](https://help.sonatype.com/en/upgrade-nexus-repository.html)
- [PostgreSQL 版本支持策略](https://www.postgresql.org/support/versioning/)
- [PostgreSQL 官方 Docker 镜像清单](https://github.com/docker-library/official-images/blob/master/library/postgres)
- [Red Hat 容器兼容性策略](https://access.redhat.com/support/policy/rhel-container-compatibility)
