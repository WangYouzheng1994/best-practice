# Docker 26.1.4 生产级安装配置手册

> **Author:** 王有政<br>
> **编码:** UTF-8<br>
> **适用系统:** CentOS 7.9 x86_64 存量服务器<br>
> **最后更新:** 2026-07-15

> CentOS 7 已于 2024-06-30 停止维护，Docker 官方也不再支持在 CentOS 7 上新装当前版本。本文固定使用 Docker 26.1.4，仅用于暂时不能迁移的存量服务器；新生产服务器优先使用受支持的 EL9 系统。

## 目录

1. [安装前确认](#1-安装前确认)
2. [安装 Docker](#2-安装-docker)
3. [配置内核参数](#3-配置内核参数)
4. [配置 daemon.json](#4-配置-daemonjson)
5. [启动 Docker](#5-启动-docker)
6. [配置私有仓库证书](#6-配置私有仓库证书)
7. [配置变更与回滚](#7-配置变更与回滚)

---

## 1. 安装前确认

执行前先完成 [CentOS 7.9 生产环境初始化](./CentOS7.9生产环境初始化手册.md)，并确认：

- `/data` 是独立数据盘；
- 服务器不存在未登记的历史 Docker、容器和数据；
- `172.30.0.0/16` 与内网、VPC、VPN、专线网段无冲突；
- 已接受 CentOS 7.9 和 Docker 26.1.4 停止维护的风险。

```bash
cat /etc/centos-release
uname -m
findmnt /data
df -hT /data
ip -4 route

rpm -qa | grep -E '^(docker|containerd|runc)' | sort
docker ps -a 2>/dev/null || true
du -sh /var/lib/docker /data/docker 2>/dev/null
```

如果 `/data` 使用 XFS，必须显示 `ftype=1`：

```bash
xfs_info /data 2>/dev/null | grep ftype
```

---

## 2. 安装 Docker

### 2.1 卸载冲突包

仅限确认没有历史业务和 Docker 数据的新装服务器：

```bash
yum remove -y \
  docker docker-client docker-client-latest docker-common \
  docker-latest docker-latest-logrotate docker-logrotate \
  docker-engine podman runc
```

不要删除 `/var/lib/docker`、`/var/lib/containerd` 或 `/data/docker`。

### 2.2 配置阿里云 Docker CE 源

如果初始化手册已经配置 Docker CE 源，不要重复添加。未配置时执行：

```bash
yum install -y yum-utils lvm2
yum-config-manager --add-repo \
  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
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

## 3. 配置内核参数

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

---

## 4. 配置 daemon.json

```bash
mkdir -p /etc/docker /data/docker
chown root:root /data/docker
chmod 0711 /data/docker

if [ -f /etc/docker/daemon.json ]; then
  cp -a /etc/docker/daemon.json \
    "/etc/docker/daemon.json.bak.$(date +%Y%m%d%H%M%S)"
fi

cat > /etc/docker/daemon.json << 'EOF'
{
  "data-root": "/data/docker",
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io",
    "https://ccr.ccs.tencentyun.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "10"
  },
  "live-restore": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
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

chown root:root /etc/docker/daemon.json
chmod 0644 /etc/docker/daemon.json
dockerd --validate --config-file=/etc/docker/daemon.json
```

主配置约定：

| 配置 | 固定值 |
| --- | --- |
| Docker 数据目录 | `/data/docker` |
| 存储驱动 | `overlay2` |
| cgroup 驱动 | `systemd` |
| 日志驱动 | `json-file` |
| 单个日志文件上限 | `100m` |
| 日志文件保留数 | `10` |
| `docker0` 网段 | `172.30.255.0/24` |
| 自定义网络地址池 | `172.30.0.0/16`，按 `/24` 分配 |

四个 `registry-mirrors` 已于 2026-07-15 完成 `library/hello-world:latest` manifest 拉取验证。公共镜像站可能变化，失效后应替换为新的可用地址，不要删除整个 `registry-mirrors` 配置。

---

## 5. 启动 Docker

```bash
systemctl daemon-reload
systemctl enable docker
systemctl start docker
```

---

## 6. 配置私有仓库证书

没有企业私有仓库时跳过本节。使用企业私有 CA 时：

```bash
REGISTRY='harbor.example.com'

mkdir -p "/etc/docker/certs.d/${REGISTRY}"
cp harbor-ca.crt "/etc/docker/certs.d/${REGISTRY}/ca.crt"
chown root:root "/etc/docker/certs.d/${REGISTRY}/ca.crt"
chmod 0644 "/etc/docker/certs.d/${REGISTRY}/ca.crt"

systemctl restart docker
```

仓库使用非默认端口时，目录名必须包含端口，例如：

```text
/etc/docker/certs.d/harbor.example.com:8443/ca.crt
```

生产仓库使用 HTTPS，不在 `daemon.json` 中配置 `insecure-registries`。

---

## 7. 配置变更与回滚

变更前备份：

```bash
mkdir -p /data/backup/docker
cp -a /etc/docker/daemon.json \
  "/data/backup/docker/daemon.json.$(date +%Y%m%d%H%M%S)"
```

修改后校验并重启：

```bash
vim /etc/docker/daemon.json
dockerd --validate --config-file=/etc/docker/daemon.json
systemctl restart docker
```

配置异常时恢复最近一次备份：

```bash
cp -a /data/backup/docker/daemon.json.<时间戳> \
  /etc/docker/daemon.json

dockerd --validate --config-file=/etc/docker/daemon.json
systemctl restart docker
```

修改 `bip` 或 `default-address-pools` 会影响现有容器网络，必须在维护窗口执行并重建相关网络。

---

## 参考资料

- [Docker Engine CentOS 安装文档](https://docs.docker.com/engine/install/centos/)
- [Docker daemon 配置参考](https://docs.docker.com/reference/cli/dockerd/)
- [Docker overlay2 配置](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)
- [Docker 日志配置](https://docs.docker.com/engine/logging/configure/)
- [CentOS Linux 7 EOL 说明](https://www.redhat.com/en/topics/linux/centos-linux-eol)
