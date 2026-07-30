# Nexus Repository 3.94.1 单体生产级 Docker 配置部署手册

> **Author:** 王有政<br>
> **Encoding:** UTF-8<br>
> **日期:** 2026-07-30<br>
> **适用系统:** CentOS 7.9 x86_64 存量服务器<br>
> **Nexus 版本:** 3.94.1（产品构建 3.94.1-06）<br>
> **Nexus 镜像:** `sonatype/nexus3:3.94.1-ubi`<br>
> **PostgreSQL 镜像:** `postgres:17.10-bookworm`

> CentOS 7.9、Docker 26.1.4 均已退出厂商支持范围。本手册只用于暂时不能迁移的存量服务器。正式接入业务前必须通过第 4 节镜像基础运行检查、完整验收和一次恢复演练；新服务器应部署在受支持的 EL9 系统上。

## 分阶段导航

| 阶段 | 使用目的 | 直接入口 |
| --- | --- | --- |
| 部署规划 | 确认架构边界、资源、路径和端口 | [1. 部署基线](#1-部署基线)<br>[2. 资源、路径与端口规划](#2-资源路径与端口规划) |
| 部署准备 | 检查宿主机并验证固定版本镜像 | [3. 部署前检查](#3-部署前检查)<br>[4. 固定镜像并执行基础运行检查](#4-固定镜像并执行基础运行检查) |
| 正式部署 | 创建持久化数据、配置并启动服务 | [5. 创建目录和密钥](#5-创建目录和密钥)<br>[6. 初始化 PostgreSQL](#6-初始化-postgresql)<br>[7. 配置 Nexus 数据源](#7-配置-nexus-数据源)<br>[8. 编写 Docker Compose 配置](#8-编写-docker-compose-配置)<br>[9. 启动服务](#9-启动服务)<br>[10. 配置 Nginx 访问入口](#10-配置-nginx-访问入口)<br>[11. 初始化 Nexus](#11-初始化-nexus)<br>[12. 配置 Maven 客户端](#12-配置-maven-客户端) |
| 运维保障 | 执行备份、恢复、升级和日常验收 | [13. 备份](#13-备份)<br>[14. 恢复](#14-恢复)<br>[15. 升级与回滚](#15-升级与回滚)<br>[16. 日常运维与验收](#16-日常运维与验收) |
| 参考资料 | 查阅产品与镜像官方文档 | [17. 官方参考](#17-官方参考) |

从第 5 节开始进入正式部署流程，操作将创建持久化目录、密钥、数据库及服务配置。

## 执行约定

- 首次部署必须按第 3 节至第 12 节的顺序执行，不要跳过中间步骤；
- 除第 11 节明确要求使用浏览器、第 12 节明确要求在 Maven 客户端执行外，所有 `bash` 命令均在 CentOS 7.9 宿主机上以 `root` 用户执行，不要先进入容器；
- `docker compose exec` 表示由 Compose 在已经运行的目标容器内执行该条子命令，执行人仍然是在宿主机终端输入命令；
- 除明确标注“仅在故障或误操作后执行”的恢复代码块外，正式部署阶段的每个 `bash` 代码块都是一个完整执行单元，应整块复制执行；确认该代码块没有报错并满足紧随其后的结果要求，再执行下一块；
- Nexus 和 PostgreSQL 只有一条正式部署路径。第 10 节根据现场条件二选一：具备域名和证书时使用“域名 + HTTPS”；只有固定公司内网 IP 且无法取得证书时使用“内网 IP + HTTP”。两个分支不得同时执行。

---

## 1. 部署基线

本手册采用以下固定结构：

- Nexus 与 PostgreSQL 由同一个 Docker Compose 项目管理；
- Nexus 使用外置 PostgreSQL，不允许以嵌入式 H2 作为本部署的生产数据库；
- PostgreSQL 只在 Compose 网络内提供服务，不发布宿主机 `5432` 端口；
- Nexus 只发布到宿主机回环地址 `127.0.0.1:18082`；
- 宿主机 Nginx 作为唯一外部入口：标准环境提供域名 HTTPS，受限内网环境提供固定 IP HTTP；
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
| TLS 证书（域名 HTTPS 分支） | `/data/module/nginx/conf/certs/${NEXUS_DOMAIN}.crt` |
| TLS 私钥（域名 HTTPS 分支） | `/data/module/nginx/conf/certs/${NEXUS_DOMAIN}.key` |

### 2.3 端口规划

| 端口 | 监听地址 | 用途 | 暴露策略 |
| --- | --- | --- | --- |
| `18083/tcp` | 宿主机业务地址 | 域名分支的 HTTP 跳转入口，或内网 IP 分支的 HTTP 业务入口 | 仅向受信研发、构建网段开放 |
| `18483/tcp` | 宿主机业务地址 | 仅用于域名 HTTPS 分支 | 仅向受信研发、构建网段开放 |
| `18082/tcp` | `127.0.0.1` | Nexus 宿主机回环入口 | 不对外开放 |
| `5432/tcp` | 仅 Compose 网络 | PostgreSQL | 不发布宿主机端口 |

域名 HTTPS 分支必须提前完成 DNS 解析并准备受信任证书。内网 IP HTTP 分支必须使用固定内网 IP，并通过网络策略限制访问来源；动态地址不得作为团队仓库地址。

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
getenforce

findmnt /data
df -hT /data
df -ih /data
free -h
nproc
timedatectl

ip -4 route
ss -lntp | grep -E ':(18082|18083|18483)[[:space:]]' || true
```

如果 `/data` 是 XFS，必须显示 `ftype=1`：

```bash
xfs_info /data | grep 'ftype=1'
```

确认以下条件后才能继续：

- `/data` 是持久化数据盘且备份目标可用；
- Docker Engine 版本为 `26.1.4`，`docker compose version` 显示 Compose v2；不得使用已停止维护的 `docker-compose` v1；
- Docker `data-root` 为 `/data/docker`；
- Docker 地址池与企业内网、VPN、VPC 和专线网段不冲突；
- `18082` 没有被其他进程占用；
- 第 10 节所选入口端口没有被其他 Nginx 站点占用；
- 服务器能够访问 Maven Central，或已配置企业允许的上游代理；
- 已确定 Nexus Community Edition 或 Pro 的使用范围并符合对应许可条款。

---

## 4. 固定镜像并执行基础运行检查

### 4.1 拉取固定版本

```bash
docker pull sonatype/nexus3:3.94.1-ubi
docker pull postgres:17.10-bookworm
```

不允许使用 `latest`。本手册固定使用 Nexus 官方 UBI + Java 21 变体；该选择不改变 CentOS 7.9 已停止支持的事实。

### 4.2 检查镜像基础运行能力

以下命令在 CentOS 7.9 宿主机上执行。命令只创建不加载生产配置、不挂载生产目录的临时容器，用于检查镜像内用户和基础运行环境，不会启动正式的 Nexus 或 PostgreSQL 服务；命令结束后，临时容器由 `--rm` 自动删除。

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

本手册后续的宿主机目录权限按照上述固定镜像中的用户标识设置。检查输出必须显示 Nexus 当前用户的 UID/GID 为 `200:200`，PostgreSQL 的 `postgres` 用户 UID/GID 为 `999:999`。

临时检查容器无法执行、Java 21 无法运行、出现 `unsupported kernel`、非法指令或 UID/GID 与预期不一致时，立即停止部署。不得在未验证数据库和升级路径的情况下直接改用旧版 Nexus。

---

## 5. 创建目录和密钥

### 5.1 创建目录

以下命令均在 CentOS 7.9 宿主机上以 `root` 用户执行，不要进入容器执行。创建的目录是后续 Docker 绑定挂载使用的宿主机目录。

```bash
install -d -m 0755 -o root -g root /data/module/nexus
install -d -m 0750 -o root -g root /data/module/nexus/compose
install -d -m 0750 -o 200  -g 200  /data/module/nexus/data
install -d -m 0700 -o 999  -g 999  /data/module/nexus/postgresql
install -d -m 0750 -o root -g root /data/module/nexus/postgresql-init
install -d -m 0700 -o root -g root /data/module/nexus/secrets
install -d -m 0700 -o root -g root /data/backup/nexus

install -d -m 0750 -o 200 -g 200 \
  /data/module/nexus/data/etc

install -d -m 0750 -o 200 -g 200 \
  /data/module/nexus/data/etc/fabric

stat -c '%u:%g %a %n' \
  /data/module/nexus/data \
  /data/module/nexus/data/etc \
  /data/module/nexus/data/etc/fabric
```

以上三个目录必须全部显示 UID/GID `200:200`、权限 `750`。`etc` 必须单独创建和授权；如果只创建 `etc/fabric`，隐式创建的中间目录 `etc` 可能属于 `root:root`，导致 Nexus 无法写入 `/nexus-data/etc/nexus.properties`。

### 5.2 生成数据库密码

```bash
(
set -Eeuo pipefail

POSTGRES_PASSWORD_FILE=/data/module/nexus/secrets/postgres-admin-password.txt
NEXUS_PASSWORD_FILE=/data/module/nexus/secrets/nexus-db-password.txt

if [ -e "${POSTGRES_PASSWORD_FILE}" ] \
  && [ -e "${NEXUS_PASSWORD_FILE}" ]; then
  echo '两个密码文件已经存在，本次不会覆盖，将校验并继续复用'
elif [ -e "${POSTGRES_PASSWORD_FILE}" ] \
  || [ -e "${NEXUS_PASSWORD_FILE}" ]; then
  echo '两个密码文件只有一个存在，状态不完整，停止部署' >&2
  exit 1
else
  install -m 0600 -o root -g root /dev/null \
    "${POSTGRES_PASSWORD_FILE}"
  install -m 0600 -o root -g root /dev/null \
    "${NEXUS_PASSWORD_FILE}"

  openssl rand -hex 32 > "${POSTGRES_PASSWORD_FILE}"
  openssl rand -hex 32 > "${NEXUS_PASSWORD_FILE}"
fi

if ! grep -Eq '^[0-9a-f]{64}$' "${POSTGRES_PASSWORD_FILE}" \
  || ! grep -Eq '^[0-9a-f]{64}$' "${NEXUS_PASSWORD_FILE}"; then
  echo '一个或两个密码文件内容无效，必须是64位十六进制字符串' >&2
  exit 1
fi

if cmp -s "${POSTGRES_PASSWORD_FILE}" "${NEXUS_PASSWORD_FILE}"; then
  echo 'PostgreSQL 管理员密码和 Nexus 数据库密码不能相同' >&2
  exit 1
fi

chown 999:999 "${POSTGRES_PASSWORD_FILE}" "${NEXUS_PASSWORD_FILE}"
chmod 0400 "${POSTGRES_PASSWORD_FILE}" "${NEXUS_PASSWORD_FILE}"

stat -c '%u:%g %a %n' \
  "${POSTGRES_PASSWORD_FILE}" \
  "${NEXUS_PASSWORD_FILE}"
)
```

两行结果必须均显示 UID/GID `999:999`、权限 `400`。该代码块不会显示密码内容；两个密码文件都已存在时会在校验通过后继续复用，不会覆盖原密码；只有一个文件存在或内容无效时会停止当前代码块，但不会退出宿主机终端。

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
(
set -Eeuo pipefail

NEXUS_DB_PASSWORD="$(tr -d '\r\n' \
  < /data/module/nexus/secrets/nexus-db-password.txt)"

if ! [[ "${NEXUS_DB_PASSWORD}" =~ ^[0-9a-f]{64}$ ]]; then
  echo 'nexus 数据库密码格式不正确，未写入数据源配置' >&2
  exit 1
fi

cat > /data/module/nexus/data/etc/fabric/nexus-store.properties << EOF
username=nexus
password=${NEXUS_DB_PASSWORD}
jdbcUrl=jdbc\:postgresql\://postgresql\:5432/nexus?currentSchema=nexus
EOF

chown 200:200 \
  /data/module/nexus/data/etc/fabric/nexus-store.properties
chmod 0600 \
  /data/module/nexus/data/etc/fabric/nexus-store.properties

unset NEXUS_DB_PASSWORD

stat -c '%u:%g %a %n' \
  /data/module/nexus/data/etc/fabric/nexus-store.properties
)
```

结果必须显示 UID/GID `200:200`、权限 `600`。密码格式不正确时，本代码块会停止且不会退出当前宿主机终端。

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

卷挂载末尾的 `:Z` 表示 Docker 为该容器设置 SELinux 私有标签。不要使用 `docker compose run nexus`，也不要使用另一个临时容器挂载 `/data/module/nexus/data` 做权限检查；这些操作可能重新标记生产目录，导致正式 Nexus 容器出现 `AccessDeniedException`。目录权限只使用第 5 节的宿主机 `stat` 命令检查，服务启动后需要进入容器时只使用 `docker compose exec`。

如果已经误执行 `docker compose run nexus` 并导致正式 Nexus 容器无法访问数据目录，仅在这种情况下执行以下恢复命令；正常首次部署不要执行 `--force-recreate`：

```bash
(
set -Eeuo pipefail
cd /data/module/nexus/compose

docker compose stop nexus
docker compose up -d \
  --no-deps \
  --force-recreate nexus
docker compose logs --since=5m --tail=300 nexus
)
```

校验 Compose 最终配置：

```bash
(
set -Eeuo pipefail
cd /data/module/nexus/compose

docker compose config --quiet
docker compose config --images
)
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
(
set -Eeuo pipefail
cd /data/module/nexus/compose

docker compose up -d postgresql

postgres_ready=0
for i in $(seq 1 60); do
  if [ "$(docker inspect --format '{{.State.Health.Status}}' \
    nexus-postgresql 2>/dev/null || true)" = 'healthy' ]; then
    postgres_ready=1
    break
  fi
  sleep 5
done

if [ "${postgres_ready}" -ne 1 ]; then
  docker compose logs --since=10m --tail=300 postgresql
  echo 'PostgreSQL 未在 5 分钟内进入 healthy 状态，停止部署' >&2
  exit 1
fi

docker compose ps
docker compose logs --since=5m --tail=200 postgresql
echo 'PostgreSQL 已进入 healthy 状态'
)
```

以上代码块会自动等待 PostgreSQL 变为 `healthy`，无需手工反复查询。看到 `PostgreSQL 已进入 healthy 状态` 后，整块执行以下数据库验收命令：

```bash
(
set -Eeuo pipefail
cd /data/module/nexus/compose

DB_OWNER="$(docker compose exec -T --user postgres postgresql \
  psql -X -v ON_ERROR_STOP=1 -At \
  -U postgres -d postgres -c \
  "SELECT pg_get_userbyid(datdba) FROM pg_database WHERE datname='nexus';")"

SCHEMA_OWNER="$(docker compose exec -T --user postgres postgresql \
  psql -X -v ON_ERROR_STOP=1 -At \
  -U postgres -d nexus -c \
  "SELECT nspowner::regrole::text FROM pg_namespace WHERE nspname='nexus';")"

EXTENSION_OWNER="$(docker compose exec -T --user postgres postgresql \
  psql -X -v ON_ERROR_STOP=1 -At \
  -U postgres -d nexus -c \
  "SELECT extowner::regrole::text || '|' || extnamespace::regnamespace::text FROM pg_extension WHERE extname='pg_trgm';")"

ROLE_FLAGS="$(docker compose exec -T --user postgres postgresql \
  psql -X -v ON_ERROR_STOP=1 -At \
  -U postgres -d postgres -c \
  "SELECT rolsuper, rolcreatedb, rolcreaterole, rolreplication, rolbypassrls FROM pg_roles WHERE rolname='nexus';")"

NEXUS_LOGIN="$(docker compose exec -T --user postgres postgresql \
  /bin/sh -ec '
    NEXUS_DB_PASSWORD="$(tr -d "\r\n" < /run/secrets/nexus_db_password)"
    export PGPASSWORD="${NEXUS_DB_PASSWORD}"
    psql -X -v ON_ERROR_STOP=1 -At \
      -h 127.0.0.1 -U nexus -d nexus \
      -c "SELECT current_user;"
  ')"

if [ "${DB_OWNER}" != 'nexus' ] \
  || [ "${SCHEMA_OWNER}" != 'nexus' ] \
  || [ "${EXTENSION_OWNER}" != 'nexus|nexus' ] \
  || [ "${ROLE_FLAGS}" != 'f|f|f|f|f' ] \
  || [ "${NEXUS_LOGIN}" != 'nexus' ]; then
  printf '%s\n' \
    "database_owner=${DB_OWNER}" \
    "schema_owner=${SCHEMA_OWNER}" \
    "extension_owner_and_schema=${EXTENSION_OWNER}" \
    "role_flags=${ROLE_FLAGS}" \
    "password_login_user=${NEXUS_LOGIN}" >&2
  echo 'PostgreSQL 初始化验收未通过，停止部署' >&2
  exit 1
fi

printf '%s\n' \
  "database_owner=${DB_OWNER}" \
  "schema_owner=${SCHEMA_OWNER}" \
  "extension_owner_and_schema=${EXTENSION_OWNER}" \
  "role_flags=${ROLE_FLAGS}" \
  "password_login_user=${NEXUS_LOGIN}" \
  'PostgreSQL 初始化验收通过'
)
```

最后一行必须显示 `PostgreSQL 初始化验收通过`。该代码块会自动确认数据库、schema 和 `pg_trgm` 扩展所有者均为 `nexus`，确认该角色没有超级用户、建库、建角色、复制或绕过行级安全权限，并使用密码通过 TCP 实际登录一次。

### 9.2 启动 Nexus

```bash
(
set -Eeuo pipefail
cd /data/module/nexus/compose

docker compose up -d nexus
docker compose ps
docker compose logs --since=5m --tail=300 nexus
)
```

首次启动通常需要数分钟。日志中的 `Custom memory flags detected` 是 Compose 已显式设置 JVM 内存参数的提示，不是报错；出现 `Starting nexus with edition COMMUNITY` 只表示正在启动，不能作为就绪依据。整块执行以下命令，以状态接口返回成功作为唯一的本机启动完成标准：

```bash
(
set -Eeuo pipefail

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
  cd /data/module/nexus/compose
  docker compose logs --since=10m --tail=300 nexus
  echo 'Nexus 未在 10 分钟内进入可写状态' >&2
  exit 1
fi

curl -fsS http://127.0.0.1:18082/service/rest/v1/status
curl -fsS http://127.0.0.1:18082/service/rest/v1/status/writable
)
```

该等待代码块失败时只停止当前代码块，不会退出宿主机终端，并会自动显示最近 10 分钟的 Nexus 日志。

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

### 9.3 当前阶段的访问方式

完成第 9 节后，Nexus 虽然已经启动，但此时还不能从其他电脑使用浏览器访问。Compose 将 Nexus 端口固定绑定为：

```text
127.0.0.1:18082:8081
```

`127.0.0.1` 是 Nexus 宿主机自身的回环地址，因此 `18082` 只接受来自宿主机本机的连接。即使外层防火墙没有限制 `18082`，其他电脑访问 `http://服务器IP:18082/` 也不会成功。

当前阶段只使用宿主机上的以下状态接口验证 Nexus，不要把端口绑定修改为 `0.0.0.0:18082`：

```bash
curl -fsS http://127.0.0.1:18082/service/rest/v1/status
curl -fsS http://127.0.0.1:18082/service/rest/v1/status/writable
```

继续完成第 10 节并选择一个 Nginx 访问分支后，才能从受信客户端浏览器访问：域名 HTTPS 分支使用 `https://正式域名:18483/`，内网 IP HTTP 分支使用 `http://固定内网IP:18083/`。首次登录和初始化操作按第 11 节执行。

---

## 10. 配置 Nginx 访问入口

### 10.1 选择访问方式

本节提供两个互斥分支，只执行符合现场条件的一个分支，然后继续执行第 10.4 节和第 10.5 节：

| 分支 | 适用条件 | 最终访问地址 | 安全说明 |
| --- | --- | --- | --- |
| A. 域名 + HTTPS | 具备正式域名和受信任证书 | `https://正式域名:18483/` | 推荐，账号密码和制品传输均由 TLS 保护 |
| B. 固定内网 IP + HTTP | 没有域名，且无法取得任何可被客户端信任的证书 | `http://固定内网IP:18083/` | 仅用于受信公司内网；账号密码和制品内容不会被加密 |

不得同时执行两个分支。分支 B 是现场条件受限时的兼容方案，不得暴露到公网、访客网络、办公无线网络或不受控终端所在网段；具备证书条件后应迁移到分支 A。

### 10.2 分支 A：域名 + HTTPS

提前将生产证书和私钥放入 `/data/module/nginx/conf/certs`。证书文件名必须是 `<正式域名>.crt`，私钥文件名必须是 `<正式域名>.key`。

以下代码块只需要修改第一行 `NEXUS_DOMAIN`，然后整块执行。保留示例域名、域名格式错误、证书缺失、证书已过期、私钥加密或证书与私钥不匹配时，代码块会在写入 Nginx 配置前停止。

```bash
(
set -Eeuo pipefail

NEXUS_DOMAIN='nexus.example.com'

if [ "${NEXUS_DOMAIN}" = 'nexus.example.com' ] \
  || ! [[ "${NEXUS_DOMAIN}" =~ ^[A-Za-z0-9][A-Za-z0-9.-]*[A-Za-z0-9]$ ]] \
  || [[ "${NEXUS_DOMAIN}" != *.* ]]; then
  echo '请把 NEXUS_DOMAIN 替换为不带协议和端口的正式域名' >&2
  exit 1
fi

CERT_FILE="/data/module/nginx/conf/certs/${NEXUS_DOMAIN}.crt"
KEY_FILE="/data/module/nginx/conf/certs/${NEXUS_DOMAIN}.key"

if [ ! -s "${CERT_FILE}" ] || [ ! -s "${KEY_FILE}" ]; then
  echo "证书或私钥不存在：${CERT_FILE} / ${KEY_FILE}" >&2
  exit 1
fi

if grep -q 'ENCRYPTED' "${KEY_FILE}"; then
  echo 'Nginx 生产私钥不能使用需要人工输入口令的加密格式' >&2
  exit 1
fi

openssl x509 -in "${CERT_FILE}" -noout -checkend 86400
openssl pkey -in "${KEY_FILE}" -noout

CERT_PUBLIC_KEY="$(
  openssl x509 -in "${CERT_FILE}" -pubkey -noout \
    | openssl pkey -pubin -outform DER \
    | sha256sum \
    | awk '{print $1}'
)"

KEY_PUBLIC_KEY="$(
  openssl pkey -in "${KEY_FILE}" -pubout -outform DER \
    | sha256sum \
    | awk '{print $1}'
)"

if [ "${CERT_PUBLIC_KEY}" != "${KEY_PUBLIC_KEY}" ]; then
  echo 'TLS 证书与私钥不匹配' >&2
  exit 1
fi

chown root:root "${CERT_FILE}" "${KEY_FILE}"
chmod 0644 "${CERT_FILE}"
chmod 0600 "${KEY_FILE}"

cat > /data/module/nginx/conf/conf.d/nexus.conf << EOF
upstream nexus_repository {
    server 127.0.0.1:18082;
    keepalive 32;
}

server {
    listen 18083;
    server_name ${NEXUS_DOMAIN};

    return 301 https://${NEXUS_DOMAIN}:18483\$request_uri;
}

server {
    listen 18483 ssl;
    http2 on;
    server_name ${NEXUS_DOMAIN};

    ssl_certificate     ${CERT_FILE};
    ssl_certificate_key ${KEY_FILE};
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

        proxy_set_header Host \$http_host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host \$host;
        proxy_set_header X-Forwarded-Port 18483;

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

openssl x509 -in "${CERT_FILE}" -noout -subject -issuer -dates
echo "Nginx 站点配置已生成：https://${NEXUS_DOMAIN}:18483/"
)
```

不得使用自签名证书作为团队生产入口。使用企业内部 CA 时，必须将根证书统一下发到开发机、构建机和 JDK 信任库。

### 10.3 分支 B：固定内网 IP + HTTP

只有在没有域名且无法取得任何可被客户端信任的证书时才使用本分支。以下代码块只需要把第一行 `NEXUS_IP` 修改为 Nexus 宿主机的固定公司内网 IPv4 地址，然后整块执行。

```bash
(
set -Eeuo pipefail

NEXUS_IP='192.0.2.10'

if [ "${NEXUS_IP}" = '192.0.2.10' ] \
  || ! [[ "${NEXUS_IP}" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; then
  echo '请把 NEXUS_IP 替换为 Nexus 宿主机的固定公司内网 IPv4 地址' >&2
  exit 1
fi

IFS=. read -r IP1 IP2 IP3 IP4 <<< "${NEXUS_IP}"
for OCTET in "${IP1}" "${IP2}" "${IP3}" "${IP4}"; do
  if [ "$((10#${OCTET}))" -gt 255 ]; then
    echo "IPv4 地址格式不正确：${NEXUS_IP}" >&2
    exit 1
  fi
done

if ! ip -4 -o addr show scope global \
  | awk '{print $4}' \
  | cut -d/ -f1 \
  | grep -Fxq "${NEXUS_IP}"; then
  echo "该 IPv4 地址不属于当前宿主机：${NEXUS_IP}" >&2
  exit 1
fi

cat > /data/module/nginx/conf/conf.d/nexus.conf << EOF
upstream nexus_repository {
    server 127.0.0.1:18082;
    keepalive 32;
}

server {
    listen ${NEXUS_IP}:18083;
    server_name ${NEXUS_IP};

    client_max_body_size 5g;
    client_body_timeout 600s;

    access_log /data/module/nginx/logs/nexus.access.log main;
    error_log  /data/module/nginx/logs/nexus.error.log warn;

    location / {
        proxy_pass http://nexus_repository;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host \$http_host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto http;
        proxy_set_header X-Forwarded-Host \$host;
        proxy_set_header X-Forwarded-Port 18083;

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

echo "Nginx 内网 IP 站点配置已生成：http://${NEXUS_IP}:18083/"
echo '警告：该入口使用 HTTP，账号密码和制品内容不会被加密'
)
```

该分支不得开放 `18483`，也不得把 `18083` 放通到整个公司网络。网络策略必须只允许实际需要使用 Nexus 的研发网段、构建网段和运维来源地址访问。

### 10.4 加载站点配置

在宿主机上以 `root` 用户整块执行。SELinux 为 `Enforcing` 时，命令会永久允许 Nginx 连接 Nexus 的 `127.0.0.1:18082`；SELinux 为 `Permissive` 或 `Disabled` 时不会修改 SELinux：

```bash
(
set -Eeuo pipefail

if [ "$(getenforce)" = 'Enforcing' ]; then
  setsebool -P httpd_can_network_connect 1
  getsebool httpd_can_network_connect
fi

/data/module/nginx/sbin/nginx -t
systemctl reload nginx
systemctl is-active --quiet nginx
echo 'Nginx 配置校验通过并已重新加载'
)
```

### 10.5 配置网络边界 （可忽略）

CentOS 7.9 初始化基线默认关闭本机 `firewalld`，不得为 Nexus 单独启用。由网络或运维人员在外层防火墙、安全组、ACL、VPN 或统一网络策略中执行：

- 域名 HTTPS 分支只允许研发网、构建网和批准的代理出口访问 `18083/18483`；
- 内网 IP HTTP 分支只允许明确的研发、构建和运维来源地址访问 `18083`，不得开放 `18483`；
- 不为 `18082` 或 `5432` 配置外部入站规则；
- 将来源网段、端口和用途登记到网络策略及资产台账；
- 需要公网接入时，由统一入口完成 WAF、TLS 和 NAT，Nexus 宿主机不得直接暴露公网。

如果现场原本启用了 `firewalld`，按《CentOS 7.9 生产环境初始化手册》的现场例外流程增加同等来源限制，不得直接使用无来源约束的 `--add-port`。

---

## 11. 初始化 Nexus

### 11.1 完成首次登录

整块执行以下命令读取一次性管理员密码：

```bash
(
set -Eeuo pipefail
cd /data/module/nexus/compose

docker compose exec nexus cat /nexus-data/admin.password
)
```

从受信客户端浏览器访问所选分支的地址。能够正常打开 Nexus 登录页面，即完成了浏览器、Nginx 和 Nexus 的实际访问验证：

```text
域名 HTTPS 分支：https://正式域名:18483/
内网 IP HTTP 分支：http://固定内网IP:18083/
```

使用 `admin` 登录并立即执行：

1. 修改管理员密码并存入企业密码管理系统；
2. 关闭 Anonymous Access；
3. 创建实名运维管理员，授予 `nx-admin`；
4. 将内置 `admin` 作为受控的应急账号，不用于日常操作；
5. 在 Base URL capability 中设置为当前浏览器使用的完整访问地址；
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

将 `__NEXUS_BASE_URL__` 替换为所选分支的浏览器访问地址，末尾不要添加 `/`：

```text
读取聚合仓库：__NEXUS_BASE_URL__/repository/maven-public/
Release 发布：__NEXUS_BASE_URL__/repository/maven-releases/
Snapshot 发布：__NEXUS_BASE_URL__/repository/maven-snapshots/
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

开发机和构建机统一使用以下模板。分发模板前，将其中全部 `__NEXUS_BASE_URL__` 替换为当前浏览器使用的 Nexus 访问地址，末尾不要添加 `/`。例如域名 HTTPS 分支替换为 `https://nexus.example.com:18483`，内网 IP HTTP 分支替换为 `http://10.10.10.20:18083`。实际账号密码通过环境变量注入：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
  <mirrors>
    <mirror>
      <id>nexus-mirror</id>
      <name>Company Nexus</name>
      <url>__NEXUS_BASE_URL__/repository/maven-public/</url>
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
          <url>__NEXUS_BASE_URL__/repository/maven-public/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>nexus-public-plugins</id>
          <url>__NEXUS_BASE_URL__/repository/maven-public/</url>
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
(
set -Eeuo pipefail

SETTINGS_FILE="${HOME}/.m2/settings.xml"
test -s "${SETTINGS_FILE}"

if grep -q '__NEXUS_BASE_URL__' "${SETTINGS_FILE}"; then
  echo 'settings.xml 中的 __NEXUS_BASE_URL__ 尚未替换' >&2
  exit 1
fi

chmod 0600 "${SETTINGS_FILE}"
echo 'settings.xml 地址占位符检查通过'
)
```

域名 HTTPS 分支禁止使用 `maven.wagon.http.ssl.insecure=true` 或关闭证书校验来绕过 TLS 问题。内网 IP HTTP 分支不提供传输加密，客户端只能从第 10.5 节限定的受信网络接入。

### 12.2 项目发布配置

项目 POM 使用与 `settings.xml` 一致的 Server ID。提交 POM 前，同样将 `__NEXUS_BASE_URL__` 替换为当前浏览器使用的 Nexus 访问地址：

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>__NEXUS_BASE_URL__/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>__NEXUS_BASE_URL__/repository/maven-snapshots/</url>
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
- 备份内容和最终加密归档的校验文件。

备份期间先优雅停止 Nexus，PostgreSQL 保持运行以执行逻辑备份。停止宽限期不得少于 120 秒。

当前脚本会先生成明文临时归档，再生成加密备份集。`/data/backup` 建议使用独立备份盘；可用空间至少应为当前 `/nexus-data` 实际占用量的 2.2 倍，避免备份过程占满 Nexus 数据盘。

### 13.2 配置备份加密公钥

备份集包含数据库角色密码哈希和 Nexus 数据库明文连接密码，必须使用企业备份公钥加密。私钥只保存在受控密码系统或灾备环境，不得存放在 Nexus 宿主机。

```bash
yum install -y gnupg2

gpg2 --import /root/nexus-backup-public.asc
gpg2 --list-keys --with-fingerprint

BACKUP_GPG_FINGERPRINT='替换为40位十六进制公钥指纹'

if [[ "${BACKUP_GPG_FINGERPRINT}" =~ ^[0-9A-Fa-f]{40}$ ]] \
  && gpg2 --batch --list-keys "${BACKUP_GPG_FINGERPRINT}" \
    >/dev/null 2>&1; then
  install -m 0600 -o root -g root /dev/null \
    /data/module/nexus/secrets/backup-gpg-recipient.txt
  printf '%s\n' "${BACKUP_GPG_FINGERPRINT}" \
    > /data/module/nexus/secrets/backup-gpg-recipient.txt
  chmod 0400 \
    /data/module/nexus/secrets/backup-gpg-recipient.txt

  gpg2 --batch --list-keys "${BACKUP_GPG_FINGERPRINT}"
else
  echo '公钥指纹未替换、格式错误或对应公钥不存在，未写入配置' >&2
fi

unset BACKUP_GPG_FINGERPRINT
```

最后一条命令必须准确显示已审批的备份公钥和指纹。

### 13.3 安装备份脚本

```bash
cat > /data/module/nexus/compose/backup.sh << 'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

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

install -d -m 0700 -o root -g root "${BACKUP_DIR}"
cd "${COMPOSE_DIR}"

docker compose stop -t 120 nexus
nexus_stopped=1

docker compose exec -T --user postgres postgresql \
  pg_dump -U postgres -d nexus -Fc \
  > "${BACKUP_DIR}/nexus.dump"

docker compose exec -T --user postgres postgresql \
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

  chmod 0600 nexus-backup.tar.gpg nexus-backup.tar.gpg.sha256

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
  if docker compose exec -T --user postgres postgresql \
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

docker compose exec -T --user postgres postgresql \
  psql -U postgres -d postgres -c \
  "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='nexus' AND pid <> pg_backend_pid();"

docker compose exec -T --user postgres postgresql \
  dropdb -U postgres --if-exists nexus

docker compose exec -T --user postgres postgresql \
  createdb -U postgres -O nexus -E UTF8 -T template0 nexus

docker compose exec -T --user postgres postgresql \
  pg_restore -U postgres -d nexus \
  --exit-on-error --no-owner --role=nexus \
  < "${BACKUP_DIR}/nexus.dump"
```

`globals.sql` 用于角色和权限核查。标准恢复由已备份的密钥及初始化脚本重建 `postgres/nexus` 角色，不直接导入 `globals.sql`；存在额外数据库角色时，由 DBA 审核后选择性恢复。

验证扩展和所有者：

```bash
docker compose exec -T --user postgres postgresql \
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
4. 预留至少 120 秒停止时间和足够数据库迁移时间；
5. Nexus 与 PostgreSQL 大版本不得在同一个变更窗口同时升级。

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

- [ ] 镜像基础运行检查通过；
- [ ] Nexus 版本为 `3.94.1-06`，使用 `3.94.1-ubi` 镜像；
- [ ] PostgreSQL 版本为 `17.10`，容器状态为 `healthy`；
- [ ] Nexus 数据源为 PostgreSQL，不是 H2；
- [ ] `/service/rest/v1/status/writable` 返回成功；
- [ ] `nexus` 是数据库、schema 和 `pg_trgm` 的所有者，但不是超级用户；
- [ ] PostgreSQL 没有宿主机端口，Nexus 只绑定 `127.0.0.1:18082`；
- [ ] 第 10 节只执行了一个访问分支，并能从受信客户端浏览器打开 Nexus 登录页面；
- [ ] 域名 HTTPS 分支的证书链、域名和有效期正确，HTTP 能跳转到 HTTPS；或内网 IP HTTP 分支只监听固定内网 IP 的 `18083`；
- [ ] 外层网络策略仅向明确的受信来源开放所选分支所需端口，未开放 `18082/5432`；
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
