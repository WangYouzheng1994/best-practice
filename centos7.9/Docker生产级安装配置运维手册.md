# Docker 26.1.4 生产级安装与使用手册

> **Author:** 王有政<br>
> **编码:** UTF-8<br>
> **适用系统:** CentOS 7.9 x86_64 存量服务器<br>
> **最后更新:** 2026-07-15

> CentOS 7 已于 2024-06-30 停止维护，Docker 官方也不再支持在 CentOS 7 上新装当前版本。本文固定使用 Docker 26.1.4，仅用于暂时不能迁移的存量服务器；新生产服务器优先使用受支持的 EL9 系统。

## 目录

1. [安装前确认](#1-安装前确认)
2. [安装 Docker](#2-安装-docker)
3. [生产基础配置](#3-生产基础配置)
4. [启动与验证](#4-启动与验证)
5. [镜像与容器常用命令](#5-镜像与容器常用命令)
6. [数据挂载](#6-数据挂载)
7. [容器网络](#7-容器网络)
8. [Docker Compose](#8-docker-compose)
9. [Dockerfile 基本规范](#9-dockerfile-基本规范)
10. [生产检查清单](#10-生产检查清单)
11. [附录：按需配置](#11-附录按需配置)

---

## 1. 安装前确认

先完成 [CentOS 7.9 生产环境初始化](./CentOS7.9生产环境初始化手册.md)，再确认以下事项：

- `/data` 是独立数据盘，Docker 数据统一放到 `/data/module/docker/data`；
- Docker 网段不能和公司内网、VPC、VPN、专线网段冲突；
- 生产镜像使用固定版本，禁止使用 `latest`；
- 已有 Docker 或业务容器的服务器，禁止直接照本文重装。

```bash
cat /etc/centos-release
uname -m
findmnt /data
df -hT /data
ip route

# 已有安装和数据检查
rpm -qa | grep -E '^(docker|containerd|runc)' | sort
docker ps -a 2>/dev/null || true
du -sh /var/lib/docker /data/module/docker/data 2>/dev/null
```

如果 `/data` 使用 XFS，必须显示 `ftype=1`：

```bash
xfs_info /data 2>/dev/null | grep ftype
```

---

## 2. 安装 Docker

### 2.1 卸载冲突的旧包

仅限确认没有历史业务和 Docker 数据的新装服务器：

```bash
yum remove -y \
  docker docker-client docker-client-latest docker-common \
  docker-latest docker-latest-logrotate docker-logrotate \
  docker-engine podman runc
```

不要删除 `/var/lib/docker`、`/var/lib/containerd` 或 `/data/module/docker/data`，历史数据是否清理由变更单单独确认。

### 2.2 配置仓库

生产环境推荐使用企业内部 Yum 仓库。没有内部仓库时，可临时使用 Docker 历史仓库：

```bash
yum install -y yum-utils lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache fast
```

### 2.3 安装 Docker

```bash
yum install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

---

## 3. 生产基础配置

### 3.1 内核参数

```bash
cat > /etc/modules-load.d/docker.conf << 'EOF'
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat > /etc/sysctl.d/99-docker.conf << 'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

### 3.2 配置 daemon.json

```bash
mkdir -p /etc/docker /data/module/docker/data
chown root:root /data/module/docker/data
chmod 0711 /data/module/docker/data

cat > /etc/docker/daemon.json << 'EOF'
{
  "data-root": "/data/module/docker/data",
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "local",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "live-restore": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
EOF

chown root:root /etc/docker/daemon.json
chmod 0644 /etc/docker/daemon.json
dockerd --validate --config-file=/etc/docker/daemon.json
```

这份配置只保留必要项：独立数据目录、`overlay2`、systemd cgroup、日志轮转和 daemon 重启时尽量保持容器运行。

不要默认写死公共 DNS 或公共镜像加速地址。内网域名应使用企业 DNS；镜像加速应使用企业 Harbor/Nexus 代理。确需配置时见文末附录。

---

## 4. 启动与验证

```bash
systemctl daemon-reload
systemctl enable --now docker

systemctl is-enabled docker
systemctl is-active docker
docker version
docker compose version
docker buildx version

docker info | grep -E \
  'Server Version|Storage Driver|Supports d_type|Cgroup Driver|Logging Driver|Docker Root Dir|Live Restore Enabled'
```

期望结果：

| 项目 | 期望值 |
| --- | --- |
| Docker Server | `26.1.4` |
| Storage Driver | `overlay2` |
| Cgroup Driver | `systemd` |
| Logging Driver | `local` |
| Docker Root Dir | `/data/module/docker/data` |
| Live Restore | `true` |

联网环境可以做一次运行测试：

```bash
docker run --rm hello-world
docker image rm hello-world:latest
```

隔离网环境换成企业镜像仓库中的测试镜像。

---

## 5. 镜像与容器常用命令

### 5.1 镜像

```bash
# 查看、拉取、删除
docker images
docker pull nginx:1.28.0
docker rmi nginx:1.28.0

# 查看镜像信息和构建历史
docker inspect nginx:1.28.0
docker history nginx:1.28.0

# 导出和导入镜像
docker save -o nginx-1.28.0.tar nginx:1.28.0
docker load -i nginx-1.28.0.tar

# 推送到私有仓库
docker tag nginx:1.28.0 harbor.example.com/base/nginx:1.28.0
docker push harbor.example.com/base/nginx:1.28.0
```

生产发布还应记录镜像 digest：

```bash
docker image inspect harbor.example.com/base/nginx:1.28.0 \
  --format '{{index .RepoDigests 0}}'
```

### 5.2 容器

```bash
# 启动示例：后台运行、自动恢复、只绑定本机端口
docker run -d \
  --name mynginx \
  --restart unless-stopped \
  -p 127.0.0.1:18080:80 \
  nginx:1.28.0

docker ps
docker ps -a
docker inspect mynginx
docker stats --no-stream mynginx

# 日志和进入容器
docker logs --tail 200 mynginx
docker logs -f --since 10m mynginx
docker exec -it mynginx /bin/sh

# 文件复制
docker cp mynginx:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./nginx.conf mynginx:/etc/nginx/nginx.conf

# 停止、启动、重启、删除
docker stop --time 30 mynginx
docker start mynginx
docker restart --time 30 mynginx
docker rm mynginx
```

进入容器只用于排查。正式修改应回到 Dockerfile、配置文件或 Compose，不能依赖容器内的临时改动。

### 5.3 空间检查

```bash
docker system df -v
du -xsh /data/module/docker/data/* 2>/dev/null | sort -h
```

禁止定时执行 `docker system prune -a --volumes`。清理前必须确认镜像、停止容器和 volume 不再用于业务或回滚。

---

## 6. 数据挂载

### 6.1 bind mount

适合由宿主机直接管理的配置和数据：

```bash
mkdir -p /data/soft/mynginx/{conf,html}

docker run -d \
  --name mynginx \
  --restart unless-stopped \
  -p 127.0.0.1:18080:80 \
  --mount type=bind,source=/data/soft/mynginx/html,target=/usr/share/nginx/html,readonly \
  --mount type=bind,source=/data/soft/mynginx/conf/nginx.conf,target=/etc/nginx/nginx.conf,readonly \
  nginx:1.28.0
```

推荐使用 `--mount`：源路径不存在时会直接报错，避免 `-v` 因拼写错误自动创建空目录。

### 6.2 volume

```bash
docker volume create myapp-data
docker volume ls
docker volume inspect myapp-data

docker run -d \
  --name myapp \
  --mount source=myapp-data,target=/app/data \
  myapp:1.0.0

docker volume rm myapp-data
```

删除 volume 前必须确认数据已经备份且没有容器引用。`docker save` 只保存镜像，不保存 volume 或 bind mount 数据。

---

## 7. 容器网络

常用模式：

| 模式 | 说明 | 建议 |
| --- | --- | --- |
| `bridge` | 默认隔离网络，通过 `-p` 发布端口 | 推荐 |
| `host` | 直接使用宿主机网络，`-p` 不生效 | 仅明确需要时使用 |
| `none` | 只有回环接口 | 特殊隔离场景 |
| `container:<名称>` | 与指定容器共享网络 | 特殊场景 |

同一应用建议使用独立网络，并通过容器名互相访问，不要依赖会变化的容器 IP：

```bash
docker network create myapp-net
docker network ls
docker network inspect myapp-net

docker run -d --name app --network myapp-net myapp:1.0.0
docker network rm myapp-net
```

`-p 18080:8080` 通常会对所有宿主机 IP 开放。生产环境优先写成 `-p 127.0.0.1:18080:8080` 或绑定明确的内网 IP，并配合外层防火墙、安全组或 ACL。

---

## 8. Docker Compose

本文已经通过 `docker-compose-plugin` 安装 Compose，统一使用新命令：

```bash
docker compose version
```

不再单独安装已经停止维护的 Python `docker-compose`。

### 8.1 Compose 示例

`/data/soft/myapp/compose.yaml`：

```yaml
name: myapp

services:
  app:
    image: harbor.example.com/prod/myapp:1.0.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:18080:8080"
    volumes:
      - ./conf:/app/conf:ro,Z
      - ./data:/app/data:Z
    cpus: 2.0
    mem_limit: 2g
    pids_limit: 256
    security_opt:
      - no-new-privileges:true
    healthcheck:
      test: ["CMD", "/app/bin/healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
```

镜像、端口、资源和健康检查命令按现场替换。生产镜像建议进一步固定 digest。

### 8.2 常用命令

```bash
cd /data/soft/myapp

docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail 200 -f app
docker compose exec app /bin/sh
docker compose restart app
docker compose stop
docker compose start
docker compose down
```

不要随意使用 `docker compose down -v`，它会删除 Compose 管理的 volume。

---

## 9. Dockerfile 基本规范

- `FROM` 使用可信基础镜像和固定版本，核心生产镜像固定 digest；
- 使用多阶段构建，不把编译工具放进运行镜像；
- 使用 `COPY`，不要滥用 `ADD`；
- 镜像内使用非 root `USER`；
- 使用 `.dockerignore` 排除 `.git`、日志、密钥和本地依赖；
- 密码和证书不能写入 `ARG`、`ENV`、`RUN` 或镜像层；
- `CMD`、`ENTRYPOINT` 使用 JSON 数组形式；
- 不使用 `docker commit` 制作生产镜像，统一由 Dockerfile 和 CI 构建；
- 上线前完成漏洞扫描，并记录镜像标签、digest 和源码提交。

构建示例：

```bash
docker buildx build \
  --file Dockerfile \
  --tag harbor.example.com/prod/myapp:1.0.0 \
  --push .
```

---

## 10. 生产检查清单

- [ ] `/data` 是独立数据盘；XFS 使用 `ftype=1`；
- [ ] Docker、containerd、Buildx、Compose 已安装，并已记录实际版本；
- [ ] `data-root` 为 `/data/module/docker/data`，日志已限制大小和数量；
- [ ] Docker 网段与内网、VPC、VPN、专线无冲突；
- [ ] 镜像来自可信仓库，未使用 `latest`，生产版本已记录 digest；
- [ ] 容器已配置重启策略、健康检查、CPU、内存和 PID 限制；
- [ ] 端口只绑定需要的宿主机 IP，并由外层网络策略限制来源；
- [ ] 数据没有放在容器可写层，业务数据已有备份和恢复验证；
- [ ] 普通业务用户未加入 `docker` 组，未暴露 Docker socket；
- [ ] 未使用 `--privileged`、公共明文仓库或不可信镜像加速；
- [ ] 已监控 Docker 服务、容器状态、重启次数、磁盘空间和 inode；
- [ ] 已制定 CentOS 7 主机迁移计划。

---

## 11. 附录：按需配置

### 11.1 自定义 Docker 网段

Docker 默认网段与企业网络冲突时，在 `/etc/docker/daemon.json` 中追加：

```json
{
  "bip": "172.30.0.1/24",
  "default-address-pools": [
    {
      "base": "172.31.0.0/16",
      "size": 24
    }
  ]
}
```

示例网段不能直接视为现场值。合并到现有 JSON 后执行：

```bash
dockerd --validate --config-file=/etc/docker/daemon.json
systemctl restart docker
docker network inspect bridge
```

修改网段会影响现有网络和容器，已有业务必须安排维护窗口并重建相关网络。

### 11.2 镜像加速与企业 DNS

有企业镜像代理时，将下面字段合并到 `daemon.json`：

```json
{
  "registry-mirrors": [
    "https://registry-proxy.example.com"
  ]
}
```

不要收集一批公共镜像站“碰运气”。默认让容器继承宿主机 DNS；必须指定时使用企业 DNS 或云厂商内网 DNS，不要写死 `8.8.8.8`。

### 11.3 私有仓库证书与登录

```bash
REGISTRY='harbor.example.com'
mkdir -p "/etc/docker/certs.d/${REGISTRY}"
cp harbor-ca.crt "/etc/docker/certs.d/${REGISTRY}/ca.crt"
chmod 0644 "/etc/docker/certs.d/${REGISTRY}/ca.crt"
systemctl restart docker

printf '%s' "${REGISTRY_PASSWORD}" | \
  docker login "${REGISTRY}" \
    --username "${REGISTRY_USERNAME}" \
    --password-stdin
```

使用最小权限机器人账号，不在文档、脚本、Compose 或 Shell 历史中写明文密码。生产仓库使用 HTTPS，不使用 `insecure-registries` 绕过证书校验。

### 11.4 SELinux 与目录权限

bind mount 权限不足时，先检查 UID/GID 和 SELinux 标签：

```bash
namei -om /data/soft/myapp/data
ls -ldZ /data/soft/myapp/data
```

目录只给一个容器使用时加 `:Z`，多个容器共享时加 `:z`。不要通过关闭 SELinux、`chmod -R 777` 或 `--privileged` 解决普通权限问题。

### 11.5 备份

需要备份的是 Compose、应用配置、镜像 digest 和业务数据，不是在线打包整个 `/data/module/docker/data`。

数据库、Redis 等组件必须使用自身备份工具生成一致性备份；普通文件数据可在停止写入后备份：

```bash
cd /data/soft/myapp
docker compose stop
tar --xattrs --acls -czf \
  "/data/backup/docker/myapp-$(date +%Y%m%d%H%M%S).tar.gz" \
  compose.yaml conf data
docker compose start
```

备份必须复制到远端，并定期做恢复演练。

### 11.6 配置变更与回滚

```bash
cp -a /etc/docker/daemon.json \
  "/data/backup/docker/daemon.json.$(date +%Y%m%d%H%M%S)"

vim /etc/docker/daemon.json
dockerd --validate --config-file=/etc/docker/daemon.json
systemctl restart docker
journalctl -u docker -n 100 --no-pager
docker ps -a
```

配置异常时恢复备份文件，再校验并重启。`live-restore` 只能降低 daemon 重启影响，不能替代业务高可用。

CentOS 7 的 Docker Engine 停留在 26.1.4。后续需要新版本时，应迁移到受支持的 EL9 主机；不要混装 EL8/EL9 RPM，也不要手工替换系统 glibc。

### 11.7 常见排查

```bash
# daemon 启动失败
dockerd --validate --config-file=/etc/docker/daemon.json
systemctl status docker --no-pager -l
journalctl -u docker -b -n 300 --no-pager

# 磁盘空间
df -hT /data
df -i /data
docker system df -v

# 网络转发和访问控制
sysctl net.ipv4.ip_forward
iptables -nvL DOCKER-USER --line-numbers
iptables -t nat -nvL DOCKER --line-numbers

# 容器状态
docker ps -a
docker inspect <container> --format \
  'status={{.State.Status}} restart={{.RestartCount}} oom={{.State.OOMKilled}}'
```

不要执行 `iptables -F` 或直接删除 `/data/module/docker/data` 内的文件。

### 11.8 参考资料

- [Docker Engine CentOS 安装文档](https://docs.docker.com/engine/install/centos/)
- [Docker 存储文档](https://docs.docker.com/engine/storage/)
- [Docker 网络文档](https://docs.docker.com/engine/network/)
- [Docker Compose 文档](https://docs.docker.com/compose/)
- [Dockerfile 文档](https://docs.docker.com/reference/dockerfile/)
- [CentOS Linux 7 EOL 说明](https://www.redhat.com/en/topics/linux/centos-linux-eol)
- [旧版 docker.md](https://share.note.youdao.com/s/I2idKVSc)
