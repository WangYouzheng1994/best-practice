# best-practice

都 3020 年了……此仓库用于沉淀服务器初始化和中间件生产部署标准，避免进入新团队后再次从基础事项重新整理。

## 文档约定

- 文档使用中文和 UTF-8 编码；作者统一为王有政。
- CentOS 7.9 相关文档统一维护在 `centos7.9/`。
- Rocky Linux 9 相关文档统一维护在 `rockylinux9/`。
- openEuler 24.03 LTS 相关文档统一维护在 `openeuler24.03/`。
- Ubuntu Server 24.04 LTS 相关文档统一维护在 `ubuntu24.04/`。
- 跨发行版的 Linux 通用文档统一维护在 `linux/`，避免操作系统初始化手册重复承载组件部署细节。
- 涉及技术决策时，在撰写过程中至少评估三个方案并确定推荐项；最终实施手册只保留已确认的结论和执行步骤，不保留方案比较过程干扰实施人员。
- 配置手册以“简单直接、结果优先、可复制执行”为准，只保留安装、最终配置、启动、备份与回滚；不写教程、方案比较、参数原理、重复验收和常见问题。
- 示例中的版本、路径、端口和密码不能直接视为现场值，上线前必须重新核验官方资料并完成环境确认。
- 目录约定：`/data` 下默认使用 `backup`、`module`、`soft`、`source` 四个顶层目录；基础组件放 `module`，Java/前端发布内容放 `soft`，源码放 `source`，本机临时备份放 `backup`。Docker `data-root` 固定使用 `/data/docker`。

## 支持的服务器操作系统

| 操作系统 | 项目目录 | 免费使用 | 国产化 | 维护状态 | 项目定位 |
| --- | --- | --- | --- | --- | --- |
| CentOS 7.9 | [`centos7.9/`](centos7.9/) | 是 | 否 | 已于 2024-06-30 结束维护 | 仅维护存量文档，不再作为新服务器选型 |
| Rocky Linux 9 | [`rockylinux9/`](rockylinux9/) | 是；商业支持另行付费 | 否 | 社区支持至 2032-05-31 | 无国产化要求时的新服务器默认选型 |
| openEuler 24.03 LTS | [`openeuler24.03/`](openeuler24.03/) | 社区版免费；商业发行版和支持另行付费 | 是 | 社区 LTS 生命周期为 4 年，可按官方机制申请延长至 6 年 | 有国产化要求且具备自主运维能力时的默认选型 |
| Ubuntu Server 24.04 LTS | [`ubuntu24.04/`](ubuntu24.04/) | 是；Ubuntu Pro 另行付费 | 否 | 提供 5 年标准安全维护，Ubuntu Pro 可延长支持周期 | 云原生、AI/GPU 和公有云场景的默认选型 |

表中的“免费使用”指操作系统镜像、软件仓库及社区更新可以免费使用，不包含厂商 SLA、现场支持、商业软件认证或信创项目所需的原厂授权。正式信创采购应以项目采购目录、软硬件兼容认证和应用厂商支持矩阵为准。

## CentOS 7.9 环境变量规范

编写或修改 `centos7.9/**/*.md` 中的软件部署手册时，如果软件通过源码包、二进制包或自定义目录安装到 `/data/module/...`，必须明确环境变量策略。

1. 团队自定义环境变量统一维护在 `/etc/profile.d/myenv.sh`，不要为单个软件创建 `/etc/profile.d/<name>.sh`。
2. 需要环境变量时，应使用当前版本稳定软链接定义 `<SOFTWARE>_HOME`，再把 `$<SOFTWARE>_HOME/bin` 加入 `PATH`。
3. 章节至少包含：编辑 `/etc/profile.d/myenv.sh`、变量内容、`chmod 644` 和 `source`。
4. systemd 服务不依赖交互式 Shell 环境变量，应在 unit 中使用绝对路径或厂商生成的明确路径。
5. 如果厂商安装器已经可靠创建 `/usr/bin` 命令入口，可以不重复修改 `PATH`。

现有示例：

```bash
export JAVA_HOME=/data/module/jdk21
export MAVEN_HOME=/data/module/maven
export MYSQL_HOME=/data/module/mysql
export REDIS_HOME=/data/module/redis7.2
```

## CentOS 7.9 专项文档

- [CentOS 7.9 生产环境初始化手册](centos7.9/CentOS7.9生产环境初始化手册.md)
- [CentOS 7.9 内网集群服务器增量配置手册](centos7.9/CentOS7.9内网集群服务器增量配置手册.md)
- [服务器端口统一规划规范](centos7.9/服务器端口统一规划规范.md)
- [Docker 26.1.4 生产级安装配置手册](centos7.9/Docker生产级安装配置运维手册.md)
- [Nginx 1.31 单体生产级配置部署手册](centos7.9/Nginx1.31单体生产级配置部署手册.md)
- [Nexus Repository 3.94.1 单体生产级 Docker 配置部署手册](centos7.9/Nexus3.94.1单体生产级Docker配置部署手册.md)
- [MySQL 8.4 单体生产级配置部署手册](centos7.9/MySQL8.4单体生产级配置部署手册.md)
- [Redis 7.2 单体生产级配置部署手册](centos7.9/Redis7.2单体生产级配置部署手册.md)
- [Redis 7.2 主从生产级配置部署手册](centos7.9/Redis7.2主从生产级配置部署手册.md)
- [Redis 7.2 Cluster 生产级配置部署手册](centos7.9/Redis7.2Cluster生产级配置部署手册.md)
- [TDengine 3.4 OSS 单体生产级配置部署手册](centos7.9/TDengine3.4OSS单体生产级配置部署手册.md)

## Linux 通用文档

- [NVM 与 Node.js 生产环境配置手册](linux/NVM与Node.js生产环境配置手册.md)
