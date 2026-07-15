# NVM 与 Node.js 生产环境配置手册

> **Author:** 王有政<br>
> **日期:** 2026-07-15<br>
> **适用系统:** Rocky Linux、AlmaLinux、RHEL、CentOS Stream 等仍在支持期内的 Linux 发行版（x86_64）<br>
> **验证版本:** NVM v0.40.5、Node.js v20.18.0（Iron，已于 2026-04-30 EOL）<br>
> **文档定位:** 本文独立说明 NVM 与 Node.js 的生产安装配置、systemd 托管和升级回滚，不在操作系统初始化手册中重复维护。

---

## 目录

1. [适用边界与生产结论](#1-适用边界与生产结论)
2. [部署前检查](#2-部署前检查)
3. [安装基础依赖](#3-安装基础依赖)
4. [创建应用用户和目录](#4-创建应用用户和目录)
5. [安装 NVM](#5-安装-nvm)
6. [安装并固定 Node.js](#6-安装并固定-nodejs)
7. [部署应用依赖](#7-部署应用依赖)
8. [使用 systemd 托管应用](#8-使用-systemd-托管应用)
9. [Node.js 升级与回滚](#9-nodejs-升级与回滚)

## 1. 适用边界与生产结论

NVM 是按 Linux 用户安装的 Node.js 版本管理工具，适合以下场景：

- 同一台构建机或应用服务器确实需要并存多个 Node.js 版本；
- 应用升级前需要保留旧版本，以便快速回滚；
- 团队已经明确接受 NVM 的按用户目录和版本绝对路径管理方式。

如果一台服务器只运行一个 Node.js 版本，优先考虑固定目录安装或容器镜像。使用 NVM 时必须遵守以下生产约定：

- NVM 和 Node.js 安装在专用应用用户下，禁止安装到 `/root/.nvm`；
- NVM、Node.js 和应用依赖全部固定精确版本，禁止部署时自动跟随 `latest` 或最新 LTS；
- systemd 使用 Node.js 可执行文件的绝对路径，禁止依赖 `.bashrc`、`.bash_profile` 或在 unit 中执行 `nvm use`；
- NVM 只管理 Node.js 版本，进程守护、开机启动、日志和资源限制由 systemd 管理；
- 正式部署前必须在同类型测试环境完成安装、启动、停止、升级和回滚验证。

### 1.1 CentOS 7.9 阻断说明

CentOS 7 已于 2024-06-30 EOL，系统自带 glibc 2.17。Node.js 新版官方 Linux 二进制包需要更高版本的 glibc，典型报错如下：

```text
version `GLIBC_2.28' not found
```

NVM 只是下载和切换 Node.js，不能解决 glibc 不兼容问题。禁止为了运行新版 Node.js 在 CentOS 7.9 上直接替换系统 glibc，这可能破坏 yum、systemd 和其他系统程序。

CentOS 7.9 存量服务器出现该问题时，应迁移到受支持的 EL8/EL9 系统或使用经过验证的容器运行时；不应把源码编译新版 Node.js 作为常规生产方案。

### 1.2 Node.js 20 生命周期说明

本文按项目约定以 Node.js `v20.18.0` 作为存量系统验证基准。Node.js 20（Iron）已于 2026-04-30 结束官方生命周期，不再接收官方安全修复。

因此，`v20.18.0` 只用于必须保持该版本兼容性的存量环境，不代表当前新建生产系统的推荐版本。新建系统或具备升级条件的系统应选择仍在官方支持期内、经过应用兼容性验证的 LTS 版本；存在安全合规要求时，Node.js 20 应视为迁移项而不是长期运行基线。

---

## 2. 部署前检查

使用 root 或具备 sudo 权限的运维用户执行：

```bash
# 发行版、EL 主版本和架构
grep -E '^(ID|ID_LIKE|VERSION_ID|PLATFORM_ID)=' /etc/os-release
rpm -E '%{rhel}'
uname -m

# 内核与 glibc
uname -r
ldd --version | head -1

# systemd
systemctl --version | head -1

# 时间同步和磁盘空间
timedatectl status
df -h /home /data 2>/dev/null || df -h /
```

### 2.1 判定是否为 EL8/EL9

`EL` 是 Enterprise Linux 兼容体系的简称。人工检查时同时查看 `/etc/os-release` 和 RPM 的 RHEL 宏：

```bash
grep -E '^(ID|ID_LIKE|VERSION_ID|PLATFORM_ID)=' /etc/os-release
rpm -E '%{rhel}'
```

常见结果如下：

| 系统 | `/etc/os-release` 关键值 | `rpm -E '%{rhel}'` |
| --- | --- | --- |
| RHEL/Rocky/AlmaLinux 8 | `VERSION_ID="8.x"`、`PLATFORM_ID="platform:el8"` | `8` |
| RHEL/Rocky/AlmaLinux 9 | `VERSION_ID="9.x"`、`PLATFORM_ID="platform:el9"` | `9` |
| CentOS Stream 8 | `ID="centos"`、`VERSION_ID="8"`、`PLATFORM_ID="platform:el8"` | `8` |
| CentOS Stream 9 | `ID="centos"`、`VERSION_ID="9"`、`PLATFORM_ID="platform:el9"` | `9` |

部署脚本推荐使用 RPM 宏判定，不要只解析发行版名称：

```bash
EL_MAJOR="$(rpm -E '%{rhel}' 2>/dev/null)"

if [[ "$EL_MAJOR" != "8" && "$EL_MAJOR" != "9" ]]; then
  echo "不支持的 EL 主版本: $EL_MAJOR" >&2
  exit 1
fi

echo "已确认当前系统为 EL${EL_MAJOR}"
```

如果 `rpm -E '%{rhel}'` 原样输出 `%{rhel}`、输出为空或命令不存在，说明当前系统不能按 RHEL 兼容发行版自动判定。此时应停止执行本手册，重新核验发行版和安装命令，不能默认当作 EL8/EL9 继续安装。

上线前确认：

| 检查项 | 要求 |
| --- | --- |
| 操作系统 | 仍在厂商支持期内 |
| 架构 | 本文示例为 `x86_64`，其他架构必须重新核验 Node.js 官方构建支持 |
| Node.js v20.18.0 系统基线 | `x86_64` 要求 Linux kernel >= 4.18、glibc >= 2.28 |
| 网络 | 可访问经过批准的 Git/NVM/Node.js 下载源，或已配置企业内网镜像 |
| 应用 | 已明确启动文件、监听端口、健康检查路径和优雅退出行为 |
| 版本 | 已确认精确的 NVM、Node.js 和包管理器版本 |

本文后续使用以下示例值，现场必须按应用实际情况核验：

| 参数 | 示例值 |
| --- | --- |
| 应用用户 | `nodeapp` |
| NVM 版本 | `v0.40.5` |
| Node.js 版本 | `v20.18.0` |
| 应用名称 | `myapp` |
| 应用目录 | `/data/soft/myapp` |
| 启动文件 | `/data/soft/myapp/dist/server.js` |
| 监听端口 | `30080` |

> **版本说明：** 示例版本是本文验证基线，不代表可以跳过兼容性测试。升级时先核验 Node.js 发布计划、安全公告、应用框架和原生依赖兼容性，再修改本文中的精确版本。

---

## 3. 安装基础依赖

EL8/EL9 系发行版执行：

```bash
sudo dnf install -y git curl ca-certificates tar xz
```

验证：

```bash
git --version
curl --version
tar --version | head -1
```

如果服务器禁止访问公网，应由企业制品库提供经过校验的 NVM Git 仓库和 Node.js 二进制镜像。不要临时关闭 TLS 校验，也不要使用来源不明的安装包。

---

## 4. 创建应用用户和目录

每个安全边界独立的应用应使用独立系统用户。本文以 `nodeapp` 为例：

```bash
sudo useradd --create-home --shell /bin/bash nodeapp

sudo install -d -o nodeapp -g nodeapp -m 0750 /data/soft/myapp
```

验证：

```bash
id nodeapp
getent passwd nodeapp
ls -ld /home/nodeapp /data/soft/myapp
```

要求：

- `nodeapp` 不加入 `wheel`，不给予 sudo 权限；
- 应用进程禁止使用 root 运行；
- 应用目录只允许发布用户和运行用户按既定流程写入；
- 应用默认监听 `1024` 以上端口，由 Nginx、HAProxy 或负载均衡器承接 `80/443`。

---

## 5. 安装 NVM

切换到应用用户：

```bash
sudo -iu nodeapp
```

从 NVM 官方仓库检出经过确认的精确标签。生产环境不直接执行未经审阅的 `curl | bash`：

```bash
git clone \
  --branch v0.40.5 \
  --depth 1 \
  https://github.com/nvm-sh/nvm.git \
  "$HOME/.nvm"

cd "$HOME/.nvm"
git describe --tags --exact-match
git remote -v
```

`git describe` 必须输出：

```text
v0.40.5
```

编辑 `/home/nodeapp/.bash_profile`，追加以下内容：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
```

加载并验证：

```bash
source "$HOME/.bash_profile"
command -v nvm
nvm --version
```

预期 `nvm --version` 输出 `0.40.5`。

> **环境变量边界：** `.bash_profile` 仅方便运维人员登录后执行 `nvm`。systemd 不读取该文件，生产服务必须使用第 8 节的绝对路径。

---

## 6. 安装并固定 Node.js

继续使用 `nodeapp` 用户执行：

```bash
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"

nvm install v20.18.0
nvm alias default v20.18.0
nvm use v20.18.0
```

验证版本和实际路径：

```bash
node --version
npm --version
nvm current
nvm which v20.18.0
readlink -f "$(command -v node)"
```

关键输出应包含：

```text
v20.18.0
/home/nodeapp/.nvm/versions/node/v20.18.0/bin/node
```

在应用仓库根目录提交 `.nvmrc`，内容只保留精确版本：

```text
v20.18.0
```

禁止在生产部署脚本中使用以下浮动写法：

```bash
# 禁止：每次部署结果可能不同
nvm install node
nvm install --lts
nvm use default
```

`default` 别名只用于交互式登录便利，不作为 systemd 的运行路径。

---

## 7. 部署应用依赖

推荐由 CI 在与生产兼容的环境中完成编译和测试，生产服务器只接收已确认的发布制品。发布内容至少包含：

- `package.json`；
- 锁文件，如 `package-lock.json`；
- 已编译的 `dist/`；
- `.nvmrc`；
- 版本号或 Git commit 等可追溯信息。

如果生产服务器需要安装运行依赖，在 `/data/soft/myapp` 下使用应用用户执行：

```bash
sudo -iu nodeapp

export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"
nvm use v20.18.0

cd /data/soft/myapp
npm ci --omit=dev
npm ls --omit=dev
```

要求：

- 必须使用锁文件和 `npm ci`，禁止用无锁定结果的 `npm install` 替代；
- 禁止执行 `sudo npm install -g ...`；
- 不要把数据库密码、令牌或私钥写入仓库、`.nvmrc`、systemd unit 或命令历史；
- 包含原生模块时，必须确认构建环境与生产系统的 CPU 架构、glibc 和 Node.js ABI 兼容；
- 安装依赖后执行应用测试和安全扫描，不能仅以 `npm ci` 成功作为上线依据。

---

## 8. 使用 systemd 托管应用

### 8.1 配置运行参数

退出 `nodeapp` 用户，使用 root 或 sudo 用户创建 `/etc/sysconfig/myapp`：

```bash
sudo vim /etc/sysconfig/myapp
```

示例内容：

```ini
NODE_ENV=production
HOST=127.0.0.1
PORT=30080
```

设置权限：

```bash
sudo chown root:nodeapp /etc/sysconfig/myapp
sudo chmod 0640 /etc/sysconfig/myapp
```

敏感配置优先从企业密钥管理系统注入。确需使用环境文件时，应严格限制读取权限，并避免通过故障输出、进程诊断或工单泄露。

### 8.2 创建 systemd unit

创建 `/etc/systemd/system/myapp.service`：

```ini
[Unit]
Description=My Node.js Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nodeapp
Group=nodeapp
WorkingDirectory=/data/soft/myapp

EnvironmentFile=/etc/sysconfig/myapp
Environment=PATH=/home/nodeapp/.nvm/versions/node/v20.18.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

ExecStart=/home/nodeapp/.nvm/versions/node/v20.18.0/bin/node /data/soft/myapp/dist/server.js

Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s
TimeoutStopSec=30s
KillSignal=SIGTERM

UMask=0027
LimitNOFILE=65535
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full

StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```

必须根据现场替换应用名称、Node.js 精确版本、应用目录和启动文件。应用还必须监听并处理 `SIGTERM`，在 `TimeoutStopSec` 内停止接收请求、完成必要清理并退出。

> **关键原则：** 不要把 `ExecStart` 写成 `nvm use ... && npm start`，也不要使用 `bash -lc` 间接加载用户 profile。绝对路径更容易审计，启动结果也不受交互式 Shell 配置影响。

校验并启动：

```bash
sudo systemd-analyze verify /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

---

## 9. Node.js 升级与回滚

### 9.1 升级流程

以下使用 `NEW_NODE_VERSION` 表示目标版本。执行时必须先把变量值替换为已经确认存在的精确版本号，禁止原样执行示例占位值。

1. 在同类型测试环境验证应用、依赖、性能、启动和优雅停止。
2. 使用应用用户安装新版本，但保留旧版本：

```bash
sudo -iu nodeapp
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"

NEW_NODE_VERSION='vXX.YY.ZZ'
nvm install "$NEW_NODE_VERSION"
nvm use "$NEW_NODE_VERSION"
node --version
```

3. 更新应用仓库中的 `.nvmrc`，重新执行 CI 构建和测试。
4. 将 `/etc/systemd/system/myapp.service` 中两处 Node.js 目录同时改为新版本：

```ini
Environment=PATH=/home/nodeapp/.nvm/versions/node/vXX.YY.ZZ/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/home/nodeapp/.nvm/versions/node/vXX.YY.ZZ/bin/node /data/soft/myapp/dist/server.js
```

5. 校验、加载并重启：

```bash
sudo systemd-analyze verify /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl restart myapp
sudo systemctl status myapp --no-pager -l
```

禁止升级后立即删除旧 Node.js 目录。应在观察期结束、确认无需回滚后再清理。

### 9.2 回滚流程

出现兼容性、性能或稳定性问题时：

1. 恢复上一版应用制品和锁文件；
2. 将 systemd unit 的 `PATH` 与 `ExecStart` 恢复为原 Node.js 绝对路径；
3. 执行校验、重载和重启；

```bash
sudo systemd-analyze verify /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl restart myapp
sudo systemctl status myapp --no-pager -l
```

Node.js 版本回滚不等于数据回滚。涉及数据库结构、消息格式或缓存结构变更时，必须执行应用发布方案中单独审核的数据兼容与回滚步骤。

---

## 官方参考

- [NVM 官方仓库](https://github.com/nvm-sh/nvm)
- [Node.js 官方下载](https://nodejs.org/en/download)
- [Node.js 发布计划](https://github.com/nodejs/release#release-schedule)
- [Node.js v20.18.0 支持平台](https://github.com/nodejs/node/blob/v20.18.0/BUILDING.md#platform-list)
- [systemd.service 文档](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)

> **Author:** 王有政 | **最后更新:** 2026-07-15
