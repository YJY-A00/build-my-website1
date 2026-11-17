# 网站搭建 Linux 操作记录

> 安全提醒：本文件曾包含数据库密码和固定弱口令示例，当前版本已脱敏。如果这些内容曾提交到远端仓库，应立即轮换相关数据库/管理员凭据，并同步更新服务器配置；只修改当前文件不能清除 Git 历史中的旧内容。

> 本记录整理了从零开始，在 Linux 服务器（Ubuntu 22.04）上搭建 WordPress 网站的过程中执行的所有 Linux 相关操作。


## 一、服务器连接与基础操作

### 1. SSH 远程连接（尝试）

- 命令：`ssh root@47.237.144.216`
- 说明：尝试通过 SSH 协议连接远程 Linux 服务器。`root` 是 Linux 系统的超级管理员账户，`@` 后面是服务器公网 IP。
- 结果：因网络问题改用 VNC 连接。

### 2. 通过 VNC（管理终端）登录

- 操作步骤：
  - 在阿里云控制台选择"远程连接" → "管理终端"
  - 输入用户名：`root`
  - 输入密码：服务器密码
- 说明：VNC 是 Linux 服务器的一种远程管理方式，不依赖网络链路，属于"带外管理"。

### 3. 切换用户（sudo -i）

- 命令：`sudo -i`
- 说明：从普通用户（`admin`）切换到超级管理员（`root`）。`sudo` 表示以管理员权限执行，`-i` 表示切换到目标用户的环境。

### 4. 查看系统信息

- 命令：
  - `uname -a`
  - `lsb_release -a`
- 说明：`uname -a` 查看 Linux 内核版本，`lsb_release -a` 查看操作系统发行版信息。

### 5. 查看系统资源占用

- 命令：
  - `top -n 1 | head -15`
  - `free -h`
  - `ps aux | grep -E "php|make|cc|gcc"`
- 说明：
  - `top -n 1 | head -15`：实时查看进程和资源占用，`-n 1` 表示只输出一次。
  - `free -h`：查看内存使用情况，`-h` 表示以人类可读格式显示。
  - `ps aux | grep -E "php|make|cc|gcc"`：列出所有进程，`grep` 过滤出特定关键词。


## 二、宝塔面板安装（Linux 环境部署）

### 6. 安装宝塔面板

- 命令：if [ -f /usr/bin/curl ];then curl -sSO https://download.bt.cn/install/install_panel.sh;else wget -O install_panel.sh https://download.bt.cn/install/install_panel.sh;fi;bash install_panel.sh ed8484bec

- 说明：
- 先用 `curl` 或 `wget` 下载宝塔安装脚本。
- `-sSO`：静默模式下载并保存。
- `-O`：指定保存文件名。
- `bash install_panel.sh`：执行安装脚本。
- `ed8484bec`：宝塔安装参数。
- 涉及的 Linux 工具：`curl`、`wget`、`bash`

### 7. 查看宝塔面板信息

- 命令：`bt default`
- 说明：`bt` 是宝塔面板的命令行管理工具，`default` 参数会显示面板登录信息。
- 安全提醒：该命令输出包含敏感登录信息，不要将终端截图、输出内容或完整面板安全入口提交到公开仓库；如已公开，应更换面板安全入口并限制管理端口来源 IP。

### 8. 启动/重启宝塔面板

- 命令：
- `bt start`
- `bt restart`
- `bt status`
- 说明：`start` 启动面板服务，`restart` 重启，`status` 查看运行状态。


## 三、LNMP 环境安装与问题排查

### 9. 编译安装 VS 极速安装

- 背景：初始服务器内存为 0.5 GiB，选择"编译安装"PHP 时，系统卡死，报错：
kcompactd0 blocked for more than 120 seconds
- 排查命令：
- `top -n 1 | head -15`
- `free -h`
- 发现：内存耗尽，`available` 小于 50MB。
- 解决方案：
1. 升级服务器内存至 1 GiB。
2. 改用"极速安装"（预编译包）。
- 说明：编译安装是从源码编译，消耗大量内存和 CPU；极速安装是下载预编译好的二进制包，对资源要求低。

### 10. 清理系统缓存

- 命令：
- `apt clean`
- `apt autoremove -y`
- 说明：`apt` 是 Ubuntu 的包管理工具，`clean` 清理下载的软件包缓存，`autoremove` 自动移除不需要的依赖包。


## 四、MySQL 数据库操作（重置密码）

### 11. 查看 WordPress 数据库配置

- 命令：`grep -E "DB_NAME|DB_USER|DB_HOST" /www/wwwroot/wordpress/wp-config.php`
- 说明：只查看数据库名称、用户名和主机字段，避免在终端记录或截图中输出数据库密码。
- 结果：数据库名称为 `wordpress`、用户名为 `wordpress`、主机为 `localhost`；密码字段已从公开文档中脱敏。

### 12. 登录 MySQL

- 命令：`mysql -u root -p`
- 说明：`mysql` 是 MySQL 的命令行客户端，`-u` 指定用户名，`-p` 提示输入密码。

### 13. 进入数据库并查看用户

- 命令：
sql
USE wordpress;
SELECT user_login, user_email FROM wp_users;
说明：USE 切换数据库，SELECT 查询用户表中的登录名和邮箱字段。

14. 重置 WordPress 管理员密码
- 安全说明：原记录中曾出现固定弱口令和直接使用 MD5 更新密码的示例，现已从公开文档中移除，不再作为操作建议。
- 建议方式：优先通过 WordPress 后台的密码重置流程，或使用受信任的 WP-CLI 在服务器上设置随机强密码；不要把密码写进命令历史、截图或 GitHub 文档。
- 凭据处理：如果旧密码曾经提交到远端仓库，应视为已经泄露并立即更换，同时更新实际配置。只修改当前文件不能清除 Git 历史中的旧内容。

15. 退出 MySQL
- 命令：
sql
EXIT;
说明：退出 MySQL 命令行界面，回到 Linux Shell。

### 五、文件与目录操作
16. 查找 WordPress 配置文件
- 命令：
bash
find / -name "wp-config.php" 2>/dev/null
说明：find 从根目录 / 开始搜索，-name 按文件名匹配，2>/dev/null 将错误输出重定向到空设备（屏蔽权限错误提示）。

结果：
bash
/www/wwwroot/wordpress/wp-config.php

17. 查看 WordPress 非敏感配置字段
- 命令：
```bash
grep -E "DB_NAME|DB_USER|DB_HOST" /www/wwwroot/wordpress/wp-config.php
```
- 说明：只检查数据库名称、用户名和主机，不要使用 `cat wp-config.php` 将包含密码的完整配置输出到终端记录或截图中。

### 六、服务器状态管理
18. 查看服务器是否卡死
现象：VNC 黑屏，按任意键无反应，系统报错：
- 命令：
bash
INFO: task kcompactd0:32 blocked for more than 120 seconds
排查：执行 top 和 free -h 查看资源占用，发现内存耗尽。
处理方式：在阿里云控制台点击“重启”或“停止再启动”，强制恢复。

19. 查看服务端口监听状态
- 命令：
bash
netstat -tlnp | grep 80
说明：netstat 查看网络连接状态，-tlnp 显示 TCP 端口监听情况，grep 80 过滤 80 端口。

作用：确认 Nginx/Apache Web 服务是否正常运行。

## Linux 命令速查表

| 命令 | 作用 |
| :--- | :--- |
| `ssh root@IP` | SSH 远程连接 Linux 服务器 |
| `sudo -i` | 切换到 root 用户 |
| `top` | 查看系统进程和资源占用 |
| `free -h` | 查看内存使用情况 |
| `ps aux` | 查看所有进程 |
| `find / -name "文件名"` | 从根目录搜索文件 |
| `cat 文件路径` | 查看文件内容 |
| `grep 关键词` | 过滤文本输出 |
| `apt update` | 更新软件包列表 |
| `apt install 软件名` | 安装软件 |
| `mysql -u root -p` | 登录 MySQL |
| `netstat -tlnp` | 查看端口监听 |
| `bt start` | 启动宝塔面板 |
| `bt restart` | 重启宝塔面板 |
| `bt status` | 查看宝塔面板状态 |
| `bt default` | 查看宝塔面板登录信息 |

## 操作记录总结

| 序号 | 操作内容 | 涉及 Linux 命令/工具 |
| :--- | :--- | :--- |
| 1 | SSH 远程连接（失败，改用 VNC） | `ssh` |
| 2 | VNC 管理终端登录 | — |
| 3 | 切换到 root 用户 | `sudo -i` |
| 4 | 查看系统资源占用 | `top`、`free`、`ps` |
| 5 | 安装宝塔面板 | `curl`、`wget`、`bash` |
| 6 | 查看宝塔面板登录信息 | `bt default` |
| 7 | 启动宝塔面板服务 | `bt start` |
| 8 | 编译安装 PHP（卡死，失败） | — |
| 9 | 内存耗尽排查与诊断 | `top`、`free -h` |
| 10 | 升级内存至 1 GiB + 改用极速安装 | — |
| 11 | 清理系统缓存 | `apt clean`、`apt autoremove` |
| 12 | 查看 WordPress 数据库配置 | `cat`、`grep` |
| 13 | 登录 MySQL | `mysql` |
| 14 | 进入数据库并查看用户表 | `USE`、`SELECT` |
| 15 | 重置管理员密码 | `UPDATE`、`MD5()` |
| 16 | 搜索 WordPress 配置文件 | `find` |
| 17 | 服务器强制重启 | 阿里云控制台 |
| 18 | 查看端口监听状态 | `netstat`、`grep` |

## 九、技术收获

通过本次 Linux 环境下的建站实践，掌握了以下技能：

| 序号 | 技能方向 | 具体内容 |
| :--- | :--- | :--- |
| 1 | Linux 基础命令 | `cd`、`ls`、`cat`、`grep`、`find`、`ps`、`top`、`free` |
| 2 | 远程管理 | SSH 连接、VNC 管理终端 |
| 3 | 用户权限管理 | `sudo -i` 切换 root 用户 |
| 4 | 包管理工具 | `apt` 安装、更新、清理软件包 |
| 5 | MySQL 数据库操作 | `USE`、`SELECT`、`UPDATE`、`EXIT` |
| 6 | 服务器故障排查 | 内存耗尽诊断、`top` 进程分析 |
| 7 | 进程管理 | `ps aux` 查看进程，`grep` 过滤 |
| 8 | 服务器重启与恢复 | 控制台重启、停止再启动 |