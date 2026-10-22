# Oracle Database 12c Release 1 在 Windows 上的安装教程

> **Author:** 王有政<br>
> **编码:** UTF-8<br>
> **适用宿主:** Windows 10 / Windows 11 x64，或 Windows Server 2012 R2 及以上<br>
> **文档定位:** 本机实验与测试环境搭建教程，不用于生产环境

本文在 Windows 上安装 Oracle Database 12c Release 1（12.1.0.2）企业版，创建一个可以直接使用的 `ORCL` 数据库，并完成监听、环境变量和远程访问配置。按顺序操作，大约需要 40 到 60 分钟，时间主要花在安装和建库上。

安装过程中最容易卡住的是第 5 步“Oracle 主目录用户选择”。本文的做法是提前创建一个 Windows 本地账户 `oracle`，密码设为 `oracle`，安装时选择“使用现有 Windows 用户”。第 3.6 节有详细说明。

> **开始之前，先确认两件事。** 这两步做错，安装会在中途失败，而且报错信息通常看不出原因：
>
> 1. **两个压缩包必须都完整解压到同一个目录。** 只解压 `1of2` 会缺少组件，安装到“复制文件”阶段报找不到文件。解压后 `setup.exe` 和 `stage\Components` 必须同时存在，核验方法见 2.5。
> 2. **`setup.exe` 必须以管理员身份运行。** 直接双击会在“先决条件检查”或服务注册阶段失败，典型表现是安装到一半回滚。启动方法见 3.1。

## 1. 最终结果与参数规划

| 项目 | 规划值 |
| --- | --- |
| 安装包 | Oracle Database 12c Release 1（12.1.0.2）Enterprise Edition，Windows x64 |
| 安装类型 | 桌面类，创建并配置数据库 |
| Oracle 基目录 | `C:\app\oracle` |
| Oracle 主目录 | `C:\app\oracle\product\12.1.0\dbhome_1` |
| Oracle 主目录用户 | Windows 本地账户 `oracle`，密码 `oracle` |
| 全局数据库名 / SID | `ORCL` |
| 字符集 | `AL32UTF8` |
| 数据库口令 | `oracle`（SYS / SYSTEM） |
| 监听端口 | `1521` |
| EM Express | `https://localhost:5500/em` |
| 内存 | 4 GB 以上，建议 8 GB |
| 磁盘 | 40 GB 以上空闲 |
| 主机名 | 纯英文，不含中文、空格和下划线 |

> 12.1.0.2 已经结束官方支持，本文只用于学习和测试。`oracle` 这种简单密码也只能出现在本机实验环境；任何对外提供服务的环境都必须换成强密码。

## 2. 安装前准备

### 2.1 环境要求

| 资源 | 最低 | 建议 |
| --- | --- | --- |
| 操作系统 | Windows 7 SP1 / Server 2008 R2 SP1 x64 | Windows 10 / 11 x64，或 Windows Server 2012 R2 及以上 |
| 内存 | 2 GB | 8 GB |
| 磁盘 | 20 GB 空闲 | 40 GB 以上空闲，SSD |
| 权限 | 管理员 | 管理员 |

Oracle 12.1 官方认证最高到 Windows 8.1 和 Windows Server 2012 R2。在 Windows 10 / 11 以及更高版本的 Windows Server 上可以安装使用，但属于非认证组合，只建议做本机实验。

安装前用管理员身份登录，确认 `C:` 盘至少有 40 GB 空闲空间。安装包和解压目录也要放在剩余空间充足的磁盘上。

### 2.2 检查主机名和网络

打开命令提示符，执行：

```cmd
hostname
ipconfig
```

主机名要求：

- 只使用英文字母、数字和短横线，不要包含中文、空格、下划线。
- 能被本机解析。执行 `ping -4 %COMPUTERNAME%`，能返回本机 IP 即可。
- 如果主机名不符合要求，先在“系统属性 → 计算机名”里修改并重启，再开始安装。建库和监听都会用到主机名，主机名有问题会导致监听注册失败。

### 2.3 创建 Oracle 主目录用户

Oracle 在 Windows 上要求用一个非管理员的本地账户来运行数据库服务，这个账户就是“Oracle 主目录用户”。本文不使用当前登录的 Windows 账户（例如 `SEMI`），而是单独创建一个专用账户：

- 用户名：`oracle`
- 密码：`oracle`

这样 Oracle 服务和数据库进程都以 `oracle` 身份运行，与日常登录账户隔离。

以管理员身份打开 PowerShell，执行：

```powershell
net user oracle oracle /add
Set-LocalUser -Name oracle -PasswordNeverExpires $true
```

也可以使用图形界面：`Win + R` 输入 `compmgmt.msc`，打开“本地用户和组 → 用户”，右键“新用户”，用户名填 `oracle`，密码填 `oracle`，取消勾选“用户下次登录时须更改密码”，勾选“密码永不过期”。

验证账户：

```cmd
net user oracle
```

确认两点：

- 账户存在且已启用。
- 账户不在 `Administrators` 组里。执行 `net localgroup Administrators` 查看，列表里不能有 `oracle`。

> 如果创建账户时提示“密码不满足密码策略”，说明本机或域启用了密码复杂度要求。把密码改成 `oracle123!@#` 这类满足复杂度的值，并且安装向导、后续登录全部使用同一个密码。不要在安装完成后再改这个账户的密码：Oracle 服务是用它登录的，改密码后服务会起不来，必须在“服务”里逐个更新登录密码。
>
> 安装完成后不要删除、禁用或重命名这个账户。

### 2.4 下载安装包

打开 Oracle 官网下载页：

<http://www.oracle.com/technetwork/cn/database/enterprise-edition/downloads/index.html>

选择 **Oracle Database 12c Release 1 (12.1.0.2) for Microsoft Windows x64**，下载两个文件：

- `winx64_12102_database_1of2.zip`
- `winx64_12102_database_2of2.zip`

两个压缩包缺一不可。

### 2.5 解压安装包

这一步最容易出错，按顺序做。

1. 新建一个纯英文、无空格的目录，例如 `D:\soft\oracle12c`。
2. 先解压 `winx64_12102_database_1of2.zip`，会得到一个 `database` 目录。
3. 再解压 `winx64_12102_database_2of2.zip`，解压位置选择**同一个** `D:\soft\oracle12c`，提示目标已存在时选择合并（覆盖）。
4. 两个包的内容最终都进入 `D:\soft\oracle12c\database`，这是正常结果。

解压完成后必须逐项核验，缺一不可：

| 检查项 | 正确结果 |
| --- | --- |
| `D:\soft\oracle12c\database\setup.exe` | 存在 |
| `D:\soft\oracle12c\database\stage` | 存在 |
| `D:\soft\oracle12c\database\stage\Components` | 存在，子目录有几十个 |
| `D:\soft\oracle12c\database\response` | 存在 |

如果只有 `setup.exe`、没有 `stage` 目录，或者 `Components` 下只有零星几个目录，说明第二个包没有解压或解压不完整。**不要在这种情况下继续安装**：向导照样能启动，但在“复制文件”阶段会因找不到组件而失败，前面的配置全部白做。

另外两个常见错误：

- 不要把两个包分别解压成 `database` 和 `database(2)` 两个目录，那相当于只装了一半。
- 解压路径不要包含中文或空格。

### 2.6 确认管理员权限

`setup.exe` 必须以管理员身份运行，否则会在检查或注册服务时失败。开始安装前，先确认当前账户是管理员：

```cmd
net localgroup Administrators
```

输出中能看到当前登录账户（例如 `SEMI`）即可。看不到的话，用管理员账户登录系统后再安装。

安装时不要用“以管理员身份运行”以外的任何方式启动 `setup.exe`，具体见 3.1。

## 3. 安装数据库

### 3.1 以管理员身份启动安装程序

打开解压目录 `D:\soft\oracle12c\database`，找到 `setup.exe`，**右键选择“以管理员身份运行”**。

不要直接双击 `setup.exe`。直接双击使用的是当前账户的普通权限，安装程序仍能启动，但会在“先决条件检查”或服务注册阶段失败，典型表现是安装到一半回滚，日志里出现权限相关的错误。

启动后确认窗口标题栏没有异常提示，点击“下一步”前可以先看第 3.8 节的检查结果，权限不足时那里会报错。

如果双击时弹出“用户账户控制”对话框，选择“是”。

如果安装程序弹出兼容性提示，选择“运行”即可。

### 3.2 配置安全更新

取消勾选“我希望通过 My Oracle Support 接收安全更新”，点击“下一步”。

弹出“未指定电子邮件地址”的提示时，选择“是”。

### 3.3 软件更新

选择“跳过软件更新”，点击“下一步”。如果提示需要 My Oracle Support 账号，直接跳过或取消。

### 3.4 安装选项

选择“创建和配置数据库”，点击“下一步”。

### 3.5 系统类

选择“桌面类”，点击“下一步”。

本文按桌面类编写，安装程序会自动创建一个 `ORCL` 数据库。如果这台机器以后要长期作为数据库服务器使用，也可以选择“服务器类”，后续选择“单实例数据库安装”和“典型安装”，其余步骤与本文相同。

### 3.6 Oracle 主目录用户选择

这一步选择“使用现有 Windows 用户”，填写：

| 字段 | 值 |
| --- | --- |
| 用户名 | `oracle` |
| 口令 | `oracle` |

填写后如果用户名右侧的黄色警告图标消失，说明校验通过，点击“下一步”。

如果提示“用户名和口令无效”或一直显示警告图标，按顺序检查：

1. Windows 里是否真的存在 `oracle` 这个本地账户。执行 `net user oracle` 确认；没有就回到 2.3 创建。
2. 密码是否就是 `oracle`，注意大小写和输入法。
3. 账户是否被禁用或锁定。执行 `net user oracle /active:yes` 启用账户。
4. 如果使用的是域账户，用户名必须写成 `域名\oracle`，不能只写 `oracle`。

关于另外两个选项：

- “创建新 Windows 用户”：由安装程序现场创建账户，但对密码有复杂度要求，`oracle` 这种简单密码通常会被拒绝。要用简单密码，就提前手工建账户，选“使用现有 Windows 用户”。
- “使用 Windows 内置账户”：Oracle 服务会以 `LocalSystem` 运行，不符合本文“主目录用户用 oracle”的要求，不选。

### 3.7 安装位置

在“安装位置”页面填写：

| 字段 | 值 |
| --- | --- |
| Oracle 基目录 | `C:\app\oracle` |
| 软件位置 | `C:\app\oracle\product\12.1.0\dbhome_1` |
| 数据库文件位置 | 保持默认（一般在 `C:\app\oracle\oradata`） |

软件位置会随基目录自动生成，一般不需要改。基目录不存在时安装程序会自动创建，不要提前在里面放其他文件。

桌面类安装会自动创建全局数据库名 `ORCL` 的数据库，不需要在这一步填写数据库名和口令。安装完成后再按 5.3 设置 SYS / SYSTEM 口令。

### 3.8 先决条件检查

等待检查完成。出现黄色警告（例如内存偏小、交换空间不足）时，可以勾选“全部忽略”，然后点击“下一步”。

如果出现红色“失败”，先按提示处理。最常见的原因是磁盘空间不足、Oracle 基目录没有权限、临时目录不可写。处理完重新检查。

### 3.9 概要

核对以下信息：

- Oracle 基目录是 `C:\app\oracle`
- Oracle 主目录用户是 `oracle`
- 全局数据库名是 `ORCL`
- 字符集是 `AL32UTF8`

确认无误后点击“安装”。

### 3.10 安装产品

进入安装阶段，整个过程通常需要 10 到 30 分钟。安装程序会依次完成“复制文件”“Oracle Net Listener 配置”“Oracle Database 配置”，最后自动建库。

期间如果 Windows 防火墙弹出提示，勾选专用网络和公用网络，点击“允许访问”，否则后面远程连不上 1521 端口。

不要点击“取消”或关闭窗口。

### 3.11 完成

看到“完成”页面后，记下页面上的 EM Express 地址，点击“关闭”。

“口令管理”按钮可以暂时不打开。安装程序已经自动创建好了 `ORCL` 数据库，SYS / SYSTEM 口令统一在 5.3 设置，保证和本文一致。

## 4. 配置环境变量

配置环境变量后，在任意命令行窗口都能直接使用 `sqlplus`、`lsnrctl` 等命令。

### 4.1 图形界面配置

1. `Win + R` 输入 `sysdm.cpl`，打开“系统属性”。
2. 切换到“高级”，点击“环境变量”。
3. 在“系统变量”中新建：

| 变量名 | 变量值 |
| --- | --- |
| `ORACLE_HOME` | `C:\app\oracle\product\12.1.0\dbhome_1` |
| `ORACLE_SID` | `ORCL` |

4. 选中系统变量 `Path`，点击“编辑”，点击“新建”，添加一行：

```text
%ORACLE_HOME%\bin
```

注意不要在路径末尾加分号。

5. 连续点击“确定”保存。

### 4.2 命令行配置

以管理员身份打开 PowerShell，执行：

```powershell
[Environment]::SetEnvironmentVariable("ORACLE_HOME", "C:\app\oracle\product\12.1.0\dbhome_1", "Machine")
[Environment]::SetEnvironmentVariable("ORACLE_SID", "ORCL", "Machine")
```

`Path` 建议用图形界面添加，避免误改系统变量。

### 4.3 验证

关闭所有命令行窗口，重新打开一个，执行：

```cmd
echo %ORACLE_HOME%
echo %ORACLE_SID%
sqlplus -v
```

能输出 Oracle 主目录路径、`ORCL` 和 SQL*Plus 版本号即可。

## 5. 验证安装

### 5.1 启动服务

Oracle 安装完成后，相关服务一般已自动启动。以管理员身份打开命令提示符：

```cmd
sc query OracleServiceORCL
sc query OracleOraDB12Home1TNSListener
```

如果状态不是 `RUNNING`，执行：

```cmd
net start OracleServiceORCL
net start OracleOraDB12Home1TNSListener
```

监听服务的实际名称可能是 `OracleOraDB12Home1TNSListener` 或 `OracleOraDB12Home2TNSListener`，以“服务”管理器中显示的名称为准。

### 5.2 检查监听

```cmd
lsnrctl status
```

输出中确认：

- 监听地址包含 `PORT=1521`。
- 状态为 `READY`。
- “服务摘要”中出现 `ORCL`。

### 5.3 登录数据库并设置口令

安装完成后先做这一步：把 SYS 和 SYSTEM 的口令统一设为 `oracle`。桌面类安装不一定会让你在向导里指定口令，装完再设可以保证结果确定。

使用操作系统认证登录：

```cmd
sqlplus / as sysdba
```

```sql
ALTER USER sys IDENTIFIED BY oracle;
ALTER USER system IDENTIFIED BY oracle;
SELECT instance_name, status FROM v$instance;
EXIT;
```

`status` 应为 `OPEN`。如果提示口令不符合复杂度要求，改成符合要求的强密码，并记住它。

再使用数据库账号密码登录一次：

```cmd
sqlplus system/oracle@localhost:1521/ORCL
```

```sql
SELECT sysdate FROM dual;
EXIT;
```

能查询出当前时间，说明监听、数据库和口令都正常。

> `sqlplus / as sysdba` 依赖 `ORA_DBA` 组，安装程序会自动把当前安装账户加进去。如果提示 `ORA-01031: insufficient privileges`，用“以管理员身份运行”的命令提示符再试。

### 5.4 打开 EM Express

浏览器访问：

```text
https://localhost:5500/em
```

用户名 `sys`，口令 `oracle`，连接身份选择 `SYSDBA`。首次打开时浏览器会提示证书不受信任，选择继续访问即可。

如果页面打不开，先用 `lsnrctl status` 确认监听正常，再执行下面的语句查看端口：

```sql
SELECT dbms_xdb_config.gethttpsport FROM dual;
```

返回 `5500` 表示端口没变。仍然打不开时，检查 Windows 防火墙是否放行了 5500 端口。

## 6. 开放远程访问

默认情况下，数据库只允许本机访问，需要放行防火墙端口。

### 6.1 添加防火墙规则

以管理员身份执行：

```cmd
netsh advfirewall firewall add rule name="Oracle-1521" dir=in action=allow protocol=TCP localport=1521
netsh advfirewall firewall add rule name="Oracle-EM-5500" dir=in action=allow protocol=TCP localport=5500
```

只开放 1521 即可满足数据库连接；5500 是 EM Express 管理页面，不需要可以不加。

### 6.2 从其他机器测试

在另一台 Windows 机器上执行：

```powershell
Test-NetConnection -ComputerName 192.168.100.10 -Port 1521
```

`TcpTestSucceeded` 为 `True` 表示端口通。

再使用 SQL*Plus 或数据库客户端连接：

```text
主机：192.168.100.10
端口：1521
服务名：ORCL
用户名：system
口令：oracle
```

如果端口通但客户端报 `ORA-12514` 或 `ORA-12505`，说明服务名写错，应使用 `ORCL`。

## 7. 日常启停、备份与卸载

### 7.1 启动和停止

启动：

```cmd
net start OracleServiceORCL
net start OracleOraDB12Home1TNSListener
```

停止时先正常关闭数据库：

```cmd
sqlplus / as sysdba
```

```sql
SHUTDOWN IMMEDIATE;
EXIT;
```

再停止服务：

```cmd
net stop OracleServiceORCL
net stop OracleOraDB12Home1TNSListener
```

不要直接强制关机或结束 Oracle 进程，容易造成数据库文件损坏。

### 7.2 快照与备份

如果 Oracle 安装在 VMware 虚拟机中，最简单可靠的备份方式是在 VMware 中对整台虚拟机打快照：

```text
虚拟机 → 快照 → 拍摄快照
```

建议在两个时间点各打一次：

- 安装前：系统初始化完成，还没有装 Oracle。
- 安装并验证完成后：得到一个可以随时回退的干净环境。

数据库层面需要导出备份时，使用 Oracle 的 `expdp` 或 RMAN，按业务需要另行配置。

### 7.3 卸载

需要重装时，必须使用 Oracle 自带的卸载程序，不要直接删除目录。

```cmd
cd /d C:\app\oracle\product\12.1.0\dbhome_1\deinstall
deinstall.bat
```

按提示确认要卸载的数据库和监听，等待卸载完成。卸载后重启，再清理残留：

- 删除 `C:\app\oracle`
- 删除 `C:\Program Files\Oracle\Inventory`
- 在“服务”中确认 Oracle 相关服务已经消失

直接删除安装目录会导致下次安装提示已有 Oracle 主目录，无法正常重装。

## 8. 常见问题

### 8.1 第 5 步主目录用户一直提示不可用

按 3.6 的四点依次检查。最常见的原因是 Windows 里根本没有 `oracle` 这个账户，或者密码输错。域账户要写成 `域名\oracle`。

### 8.2 先决条件检查出现失败

黄色警告可以勾选“全部忽略”继续。红色失败一般是内存不足、磁盘空间不足、基目录权限不对或临时目录不可写，按提示处理后重新检查。

### 8.3 安装到一半失败

先确认是不是下面两个原因，这两种情况占了大多数：

1. `setup.exe` 不是以管理员身份运行的。按 3.1 重新以管理员身份启动。
2. 两个压缩包没有完整解压到同一目录。按 2.5 核验 `stage\Components` 是否存在且内容完整，缺了重新解压。

再看安装日志：

```text
C:\Program Files\Oracle\Inventory\logs\installActions*.log
```

失败后先执行 7.3 的卸载流程，再重新安装。直接删除目录后重装通常会再次失败。

### 8.4 监听启动失败或 1521 端口被占用

```cmd
netstat -ano | findstr :1521
```

最后一列是占用端口的进程 PID，在任务管理器中结束该进程，或者用 `lsnrctl start` 查看具体报错。

### 8.5 客户端报 ORA-12541、ORA-12514、ORA-12505

- `ORA-12541`：监听没有启动，或防火墙没有放行 1521。
- `ORA-12514` / `ORA-12505`：服务名写错。连接串统一使用 `ORCL`。
- 在数据库本机执行 `ALTER SYSTEM REGISTER;`，再回到客户端重试。

### 8.6 报 ORA-01017 用户名/口令无效

先确认输入的是 `system` / `oracle`，注意大小写和输入法。如果确实忘记了 SYS 密码，用管理员命令提示符登录后重置：

```cmd
sqlplus / as sysdba
```

```sql
ALTER USER sys IDENTIFIED BY oracle;
ALTER USER system IDENTIFIED BY oracle;
EXIT;
```

### 8.7 卸载后重装提示已有 Oracle 主目录

说明上次卸载不完整。使用 `deinstall.bat` 重新卸载，清理 `C:\app\oracle` 和 `C:\Program Files\Oracle\Inventory`，重启后再安装。
