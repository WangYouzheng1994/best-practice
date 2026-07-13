# best-practice

都 3020 年了……此仓库用于沉淀服务器初始化和中间件生产部署标准，避免进入新团队后再次从基础事项重新整理。

## 文档约定

- 文档使用中文和 UTF-8 编码；作者统一为王有政。
- CentOS 7.9 相关文档统一维护在 `centos7.9/`。
- 涉及技术决策时，在撰写过程中至少评估三个方案并确定推荐项；最终实施手册只保留已确认的结论和执行步骤，不保留方案比较过程干扰实施人员。
- 组件部署手册应引用服务器初始化和统一端口规划规范，包含验证、备份、升级、回滚与风险说明。
- 示例中的版本、路径、端口和密码不能直接视为现场值，上线前必须重新核验官方资料并完成环境确认。
- 目录约定：`/data` 下固定保留 `backup`、`module`、`soft`、`source` 四个顶层目录；基础组件放 `module`，Java/前端发布内容放 `soft`，源码放 `source`，本机临时备份放 `backup`，组件不得新增其他顶层目录。

## CentOS 7.9 环境变量规范

编写或修改 `centos7.9/**/*.md` 中的软件部署手册时，如果软件通过源码包、二进制包或自定义目录安装到 `/data/module/...`，必须在安装验证之后检查环境变量策略，并在手册中明确说明。

1. 团队自定义环境变量统一维护在 `/etc/profile.d/myenv.sh`，不要为单个软件创建 `/etc/profile.d/<name>.sh`。
2. 需要环境变量时，应使用当前版本稳定软链接定义 `<SOFTWARE>_HOME`，再把 `$<SOFTWARE>_HOME/bin` 加入 `PATH`。
3. 章节至少包含：编辑 `/etc/profile.d/myenv.sh`、变量内容、`chmod 644`、`source`、`which`/`--version` 验证。
4. systemd 服务不依赖交互式 Shell 环境变量，应在 unit 中使用绝对路径或厂商生成的明确路径。
5. 如果厂商安装器已经可靠创建 `/usr/bin` 命令入口，可以不重复修改 `PATH`，但手册必须记录该决策，并验证 `which`、`readlink -f` 和版本输出。

现有示例：

```bash
export JAVA_HOME=/data/module/jdk21
export MYSQL_HOME=/data/module/mysql
export REDIS_HOME=/data/module/redis7.2
```
