# 第二次部署记录：开学后重新搭建网站

## 1. 本次实践背景

暑假首次使用的域名到期后，开学后我重新申请了云服务器，并重新安装、配置相关应用，再次搭建同一个 Go + MySQL + Nginx 个人网站。

这次实践沿用原有技术路线，但从一台全新的 Ubuntu 服务器开始重新准备环境。与第一次相比，本次记录更强调从系统初始化、基础工具安装到服务部署的完整过程。

## 2. 当前已确认的环境信息

| 项目 | 当前已知信息 |
| --- | --- |
| 云服务器 | 阿里云轻量应用服务器（具体规格待补充） |
| 操作系统 | Ubuntu（截图中未展示具体小版本；此前项目记录为 Ubuntu 24.04，建议登录服务器核实） |
| Web 服务 / 反向代理 | Nginx，通过 apt 安装 |
| 数据库 | MySQL Server，通过 apt 安装；目标版本为 MySQL 8.0 |
| 后端语言 | Go，截图采用 Go 1.23.4 Linux amd64 安装包作为示例 |
| Go 模块代理 | `https://goproxy.cn,direct` |
| 前端 | HTML / CSS / JavaScript（沿用原项目） |
| 项目架构 | Nginx + Go + MySQL |

> 版本说明：截图中的 Go 1.23.4 是当时文档命令所使用的历史版本示例，不代表当前推荐版本。新部署应先查看 [Go 官方下载页](https://go.dev/dl/) 选择仍受支持、且与项目 `go.mod` 兼容的版本；服务器实际安装版本以 `go version` 输出为准。Ubuntu 的具体版本也请以 `lsb_release -a` 的结果为准。

## 3. 第二次部署过程

以下按截图中整理的实际操作顺序记录。命令中的用户名、数据库名和目录名属于示例时，应以服务器实际配置为准。

### 步骤 0：系统初始化与基础工具安装

拿到新的 Ubuntu 云服务器后，先更新软件源和已安装软件，再安装网络下载、版本管理、解压、编辑及防火墙工具。

```bash
# 更新软件源列表并升级已安装软件
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y curl wget git unzip vim ufw

# 放行 SSH、HTTP、HTTPS 端口（22 必须与 sshd 实际监听端口一致）
sudo ufw allow 22/tcp    # SSH；若使用非标准端口，改为实际端口
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS

# 启用前检查已添加规则；确认 SSH 规则正确后再启用
sudo ufw show added
sudo ufw enable

# 启用后核对生效状态
sudo ufw status verbose
```

启用防火墙前，应先确认 SSH 实际监听端口，并确保该端口已放行；如果 SSH 使用非标准端口，不能只放行 22。最好保留云厂商 VNC/管理终端作为回退通道。云厂商安全组与 UFW 是两层独立控制，也要分别检查。此架构通常只需对公网开放 SSH、80、443；MySQL 的 3306 和 Go 的 8080 不应为了方便而直接开放。

### 步骤 1：安装并启动 Nginx

使用 Ubuntu 软件源安装 Nginx，并设置为开机启动、立即启动。

```bash
# 安装 Nginx
sudo apt install -y nginx

# 设置开机自启并立即启动
sudo systemctl enable --now nginx

# 检查 Nginx 运行状态
sudo systemctl status nginx
```

**验证方式：** 在浏览器中访问服务器公网 IP。如果出现 Nginx 默认欢迎页（通常显示 “Welcome to nginx!”），说明 Nginx 已能够对外提供默认页面。该验证只说明 Web 服务入口可访问，不代表网站前后端已经部署完成。

### 步骤 2：安装并初始化 MySQL

通过 apt 安装 MySQL Server，设置开机自启并检查服务状态。

```bash
# 安装 MySQL 服务端
sudo apt install -y mysql-server

# 设置开机自启并立即启动
sudo systemctl enable --now mysql

# 检查 MySQL 运行状态
sudo systemctl status mysql
```

随后进入 MySQL 命令行，创建项目数据库和仅供本机连接的应用账户，并授予该账户项目数据库的权限。

```sql
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'myapp'@'localhost' IDENTIFIED BY '请替换为强密码';

GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'myapp'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

说明：

- `myapp` 是截图中的示例数据库名和应用用户名；如实际项目使用其他名称，应保持 Go 配置与 MySQL 配置一致。
- 密码必须替换为独立的强密码。**不要把真实密码、可用密码或包含真实凭据的命令提交到 GitHub。**
- 不要因为担心 DSN 解析而把密码限制为简单字母和数字。go-sql-driver/mysql 文档说明密码可以包含任意字符；如果连接失败，应检查 DSN 的拼接方式、Shell 引号、环境变量读取和实际错误信息，而不是降低密码复杂度。建议从权限受限的环境变量或 Systemd 环境文件读取凭据，不要把密码写入源码、命令历史或 GitHub。
- 应用账户应只授予业务运行所需权限。上面的 CRUD 权限适合常见读写场景；若应用需要建表或执行数据库迁移，应使用单独的迁移账户，避免长期运行账户拥有 DDL 权限。
- MySQL 账户的 Host 必须与 Go 的连接方式匹配：若 DSN 通过 Unix Socket 连接，可使用 `'myapp'@'localhost'`；若通过 TCP 连接到 `127.0.0.1:3306`，应按实际情况创建并授权 `'myapp'@'127.0.0.1'`。两种方式都只允许本机访问，不需要开放公网 3306。

### 步骤 3：安装 Go 后端语言环境

截图采用 Go 官方 Linux amd64 二进制压缩包安装方式，而不是直接使用 Ubuntu 软件源中的 Go 包。示例版本为 Go 1.23.4。

```bash
# 进入临时目录并下载 Go 安装包（示例版本：1.23.4）
cd /tmp
wget https://go.dev/dl/go1.23.4.linux-amd64.tar.gz

# 如果已有 Go 安装，先核对当前命令、版本和实际安装路径
which go
go version
readlink -f "$(command -v go)"

# 若确认 /usr/local/go 是需要替换的旧安装，先备份/移走它，确保备份目标不存在
# sudo mv /usr/local/go /usr/local/go.backup-$(date +%F)
# 不要将新压缩包直接解压到已有 /usr/local/go 目录中

# 下载后先核对 SHA256：将结果与 Go 官方下载页公布的校验值比较
sha256sum go1.23.4.linux-amd64.tar.gz

# 解压到 /usr/local
sudo tar -C /usr/local -xzf go1.23.4.linux-amd64.tar.gz

# 建立全局命令软链接
sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go
sudo ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt

# 验证 Go 是否安装成功
go version
```

> 注意：`sudo rm -rf /usr/local/go` 会删除该目录下已有的 Go 安装。只有确认该目录确实用于旧版 Go、且不需要保留时才执行。下载链接中的版本号和架构也应与实际目标环境匹配。

为解决国内服务器拉取 Go 模块依赖时连接默认代理超时的问题，配置 Go 模块代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct

# 查看配置结果
go env GOPROXY
```

截图中使用的是七牛云国内 Go 模块代理地址。配置后，Go 获取模块依赖时会优先使用该代理，并在需要时按配置回退到 direct。

### 步骤 4：上传 Go 项目代码到服务器

截图给出了两种方式：本地通过 `scp` 直接上传，或先将代码推送到 GitHub/Gitee，再在服务器上克隆。示例服务器 IP 为 `8.133.191.82`，项目目录为 `/opt/myapp`；这些是截图中的示例值，实际使用时应替换为当前服务器信息。

**方式一：使用 scp 从本地上传**

先在服务器上创建项目目录：

```bash
ssh root@8.133.191.82 "mkdir -p /opt/myapp"
```

再在本地 Windows PowerShell 或其他终端中执行：

```bash
scp -r ./你的本地Go项目目录/. root@8.133.191.82:/opt/myapp/
```

**方式二：通过 Git 克隆**

先在本地将代码推送到 GitHub 或 Gitee，再登录服务器执行：

```bash
cd /opt
git clone https://github.com/你的用户名/你的仓库名.git myapp
cd myapp
```

如果仓库是私有仓库，需要先配置服务器的 Git 身份验证方式。不要把访问令牌或私钥写入公开文档。

### 步骤 5：编译 Go 后端

进入项目目录，整理依赖并编译 Linux 可执行文件。截图中的输出文件名为 `myapp`。

```bash
cd /opt/myapp

# 根据已提交的 go.mod / go.sum 下载依赖
go mod download

# 编译为 Linux 可执行文件
go build -o myapp .

# 添加可执行权限
chmod +x myapp

# 建议由 root 管理程序文件，服务账户只获得读取和执行权限
sudo chown -R root:root /opt/myapp
sudo chmod 755 /opt/myapp
sudo chmod 755 /opt/myapp/myapp
```

> 编译前应确认服务器 Go 环境与项目依赖兼容，且项目入口包位于当前目录（否则需调整 `go build` 的目标路径）。`go mod tidy` 主要用于整理依赖声明，可能改动 `go.mod` 和 `go.sum`；部署时通常使用已审核并提交的依赖文件执行 `go mod download`，如确需 tidy，应检查差异后再提交。
>
> 截图中记录的 `chown -R www-data:www-data /opt/myapp` 不建议直接用于生产环境：Web 服务账户不应拥有源码和可执行文件的写权限。可让程序文件归 root 管理、目录和二进制保持可读/可执行，再由 Systemd 以低权限用户运行。若程序需要写文件，应单独建立数据目录并只授予该目录写权限。

### 步骤 6：使用 Systemd 管理 Go 后端

截图通过 Systemd 将 Go 程序配置为后台服务，使其能够在系统启动时运行，并在进程异常退出时自动重启。

如果 Go 程序通过环境变量读取数据库连接信息，可在服务器本机创建一个不提交到 Git 的环境文件。下面的变量名只是示例，应与程序实际读取的变量保持一致：

```bash
sudo install -d -m 0755 /etc/myapp
sudo touch /etc/myapp/myapp.env
sudo chown root:root /etc/myapp/myapp.env
sudo chmod 600 /etc/myapp/myapp.env
sudoedit /etc/myapp/myapp.env
```

文件内容示例（不要把真实值写入仓库或截图）：

```ini
DB_USER=myapp
DB_PASSWORD=请替换为随机强密码
```

创建服务文件：

```bash
sudo nano /etc/systemd/system/myapp.service
```

写入以下配置：

```ini
[Unit]
Description=Go Web App
Wants=network-online.target
After=network-online.target mysql.service

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/myapp.env
ExecStart=/opt/myapp/myapp
Restart=on-failure
RestartSec=3

# 可选的基础安全加固；部署前确认程序不依赖受限制的目录
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

保存后执行：

```bash
# 重新加载 Systemd 配置
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable myapp

# 启动服务
sudo systemctl start myapp

# 查看服务状态
sudo systemctl status myapp
```

**验证方式：** 查看服务状态，确认出现 `active (running)`。如果启动失败，可使用以下命令查看服务日志：

```bash
sudo journalctl -u myapp -n 100 --no-pager
```

> `After=mysql.service` 只表示启动顺序，不等于数据库已经可以接受连接，也不代表应用具备数据库就绪重试能力。Go 程序应对数据库连接失败进行有限重试或在启动阶段明确报错；还要核对服务用户是否能读取所需配置文件。环境文件应只保留在服务器上并限制权限；如果程序不读取环境变量，应删除 service 文件中的 `EnvironmentFile` 行。

### 步骤 7：配置 Nginx 静态资源与 API 反向代理

截图采用的架构是：Nginx 直接提供前端静态文件，将以 `/api/` 开头的请求转发给监听在本机 `127.0.0.1:8080` 的 Go 后端。

创建站点配置文件：

```bash
sudo nano /etc/nginx/sites-available/myapp
```

截图中的配置示例如下：

```nginx
server {
    listen 80;
    server_name 你的新域名 www.你的新域名;

    # 前端静态资源目录
    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 请求反向代理到 Go 后端
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

启用站点并检查配置：

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp

# 移除默认站点（仅在确认不再需要时执行）
sudo rm -f /etc/nginx/sites-enabled/default

# 检查 Nginx 配置语法
sudo nginx -t

# 配置检查通过后平滑重载
sudo systemctl reload nginx
```

注意事项：

- `root /var/www/myapp` 是前端静态文件目录，需要先将前端文件部署到该目录；它与后端项目目录 `/opt/myapp` 不同。
- `proxy_pass` 的端口必须与 Go 程序实际监听端口一致。截图使用的是 `8080`。
- 当前写法 `proxy_pass http://127.0.0.1:8080;` 没有在端口后添加 URI，通常会把原始请求 URI（包括 `/api/` 前缀）传给 Go；如果后端路由并不包含此前缀，需根据实际路由调整 `proxy_pass` 的尾部斜杠，并用真实 API 路径验证，不能只凭配置语法通过就判断联调成功。
- Go 后端应监听本机回环地址（例如 `127.0.0.1:8080`），避免在不需要时直接暴露后端端口。
- `server_name` 应替换为当前真实域名，不要继续沿用已经过期的旧域名。
- 如果前端不是单页应用，需根据实际路由调整 `try_files` 规则。

### 步骤 8：绑定新域名并配置 HTTPS

截图文字将“域名已完成 ICP 备案”作为前提，但当前仓库没有备案状态凭证，因此只能记为截图中的前提，不能据此确认备案已经完成。备案要求应根据服务器接入地、服务内容及主管部门/接入服务商的现行规则核实；DNS 解析和 HTTPS 证书都不能替代备案手续。域名和服务器信息应以本次实际使用的为准。

**第一步：修改 Nginx 域名配置**

将站点配置中的 `server_name` 改为新域名，例如：

```nginx
server_name yourdomain.com www.yourdomain.com;
```

修改后检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**第二步：先检查 DNS，再安装 Certbot 和 Nginx 插件**

确认域名的 A 记录指向本次服务器公网 IP；如果配置了 AAAA 记录，也要确认 IPv6 指向正确。可使用以下命令检查解析结果（没有 `dig` 时需先安装对应 DNS 工具）：

```bash
dig +short yourdomain.com A
dig +short www.yourdomain.com A
dig +short yourdomain.com AAAA
```

同时确认云安全组和 UFW 允许公网访问 TCP 80、443，且 Nginx 站点配置已通过 `nginx -t`。

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

**第三步：申请证书并让 Certbot 配置 Nginx**

将命令中的示例域名替换为自己的真实域名：

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

按提示填写邮箱、同意服务条款，并根据实际需求选择是否将 HTTP 自动重定向到 HTTPS。申请前应确认域名 DNS 已正确解析到该服务器，且公网 80/443 端口可访问。

**第四步：测试证书自动续期**

```bash
sudo certbot renew --dry-run
```

截图说明 Let's Encrypt 证书有效期为 90 天，Certbot 可通过定时任务自动尝试续期。执行 dry-run 是对续期流程的测试，不等于已经验证未来每次续期都必然成功。可进一步检查当前证书和续期定时器：

```bash
sudo certbot certificates
systemctl list-timers --all | grep -i certbot
```

## 4. 关键配置关系

本次部署截图体现的请求链路如下：

```text
用户浏览器
    |
    | HTTP / HTTPS
    v
Nginx（80 / 443）
    |
    |-- 静态页面请求 --> /var/www/myapp
    |
    |-- /api/ 请求 --> 127.0.0.1:8080
                           |
                           v
                       Go 后端
                           |
                           v
                       MySQL
```

其中，截图中的后端服务目录为 `/opt/myapp`，前端静态目录为 `/var/www/myapp`，Go 服务端口为 `8080`。这些值属于本次操作说明中的配置示例，最终应以服务器实际配置和验证结果为准。

## 5. 上线验收清单

- [ ] SSH 可以正常连接，防火墙规则不会阻断管理连接。
- [ ] Nginx 服务正常运行，公网 IP 可以访问默认页或已部署站点。
- [ ] MySQL 服务正常运行，项目应用账户可以连接目标数据库。
- [ ] `go version` 显示预期的 Go 版本。
- [ ] `go env GOPROXY` 显示预期的模块代理配置。
- [ ] Go 项目代码已上传或克隆，依赖整理和编译成功。
- [ ] Systemd 服务状态为 `active (running)`，并能查看运行日志。
- [ ] 前端文件已部署到 Nginx 配置的静态目录。
- [ ] Nginx 配置检查通过，API 反向代理能够访问 Go 后端。
- [ ] 新域名 DNS 解析指向当前服务器。
- [ ] HTTPS 证书申请成功，浏览器访问无证书错误。
- [ ] `certbot renew --dry-run` 测试通过。
- [ ] 服务器重启后，相关服务能够按预期恢复。

## 6. 与第一次部署的对比

| 对比维度 | 第一次部署 | 第二次部署 |
| --- | --- | --- |
| 服务器 | 暑假使用的旧服务器环境 | 重新申请的云服务器 |
| 系统准备 | 以第一次部署记录为准 | 从 Ubuntu 基础环境重新初始化 |
| Nginx | 完成网站 Web 服务配置 | 使用 apt 安装并设置开机自启 |
| MySQL | 完成项目数据库配置 | 使用 apt 安装 MySQL Server 并创建项目数据库账户 |
| Go | 完成后端运行环境配置 | 使用 Go 官方 Linux amd64 压缩包安装，并配置模块代理 |
| 项目上传 | 以第一次部署记录为准 | 记录 scp 上传和 Git 克隆两种方式 |
| 后端进程 | 以第一次部署记录为准 | 使用 Systemd 管理 Go 程序 |
| 前后端联调 | 以第一次部署记录为准 | Nginx 静态资源托管并将 /api/ 反向代理到 Go 服务 |
| 域名 | 暑假申请的域名，后来到期 | 重新申请域名，具体信息待补充 |
| 备案与 HTTPS | 以第一次部署记录为准 | 记录 DNS、Certbot 证书申请与续期测试流程 |

## 7. 文档核对与待验证事项

本次只对仓库中的部署说明、命令顺序和配置关系进行了静态审查，没有连接服务器执行命令。因此，下列项目仍需在实际服务器上核实：

- [ ] Ubuntu 具体版本、MySQL 实际版本和 Go 实际版本。
- [ ] SSH 实际监听端口，以及 UFW 和云安全组的最终规则。
- [ ] MySQL 应用账户的 Host 与 Go DSN 连接方式是否匹配。
- [ ] Go 程序实际监听地址、端口、健康检查接口和数据库重试行为。
- [ ] Systemd 服务是否真实达到 `active (running)`，重启后能否恢复。
- [ ] Nginx 静态目录、API URI 转发结果和真实接口响应。
- [ ] 域名 A/AAAA 解析、备案状态、证书签发和续期测试结果。

以上未打勾项目不能在项目介绍中表述为“已验证”或“部署成功”。

## 8. 本次复盘

第二次部署从一台新的 Ubuntu 服务器开始，重新完成系统更新、基础工具安装、防火墙规则配置，以及 Nginx、MySQL 和 Go 运行环境的准备；随后将 Go 项目部署到服务器，编译为 Linux 可执行文件，并通过 Systemd 管理后端进程；最后使用 Nginx 提供静态资源和 API 反向代理，并通过 Certbot 申请 HTTPS 证书。

这份记录不仅保留了部署命令，也梳理了各组件之间的关系：Nginx 作为统一 Web 入口，Go 负责后端 API 逻辑，MySQL 保存业务数据，Systemd 负责后端进程管理，Certbot 协助管理 HTTPS 证书。

需要说明的是，本文根据操作截图整理，命令和配置中的 IP、域名、路径、用户名、端口均应以实际服务器为准。正式用于项目展示前，还应补充真实的服务状态、接口访问、HTTPS 访问和重启验证结果，避免将操作指南误写成未经验证的上线结论。


## 9. 部署变更前备份与回滚建议（运维参考）

> 以下是通用运维建议，不代表本次部署已执行。数据库备份和环境配置可能包含敏感信息，只能保存在权限受限的位置；建议另存加密副本，不要上传 GitHub。

### 备份数据库

在 Ubuntu 默认的 MySQL root 本地 Socket 认证可用时，可用 root 权限执行备份（如果服务器的认证方式不同，应先核对实际认证配置）：

```bash
# 创建仅 root 可访问的备份目录
sudo install -d -m 700 /var/backups/myapp

# 备份 myapp 数据库；InnoDB 表适用单事务备份
sudo bash -c 'umask 077; mysqldump --single-transaction --routines --triggers --events myapp > /var/backups/myapp/myapp-$(date +%F-%H%M%S).sql'

# 检查备份文件是否生成且大小正常
sudo ls -lh /var/backups/myapp/
```

- `--single-transaction` 主要为 InnoDB 表提供一致性快照；非事务型表不能因此保证一致性。
- 备份文件包含数据库数据，应限制访问、加密并复制到独立存储。仅保存在同一服务器不能防止服务器故障或误删。
- 备份是否可用，最终应通过隔离环境的恢复测试验证，而不是只检查文件存在。

### 保存应用与配置快照

发布或修改配置前，按实际路径保存旧版 Go 可执行文件、前端静态文件、Nginx 站点配置、Systemd 服务文件及环境文件。环境文件可能包含密码，必须保持严格权限并采用安全存储方式。保留版本和日期，避免直接覆盖唯一备份。

### 回滚思路

1. 确认故障范围，优先回退到已保存的应用二进制文件、前端文件和配置版本。
2. 回退 Systemd 或 Nginx 配置后，先执行 `sudo systemctl daemon-reload`（涉及 Systemd 时）及 `sudo nginx -t`（涉及 Nginx 时），通过检查后再重启/重载服务。
3. 验证服务状态、首页和实际 API 路径，检查日志。
4. 如果发布涉及数据库结构迁移，回退应用代码不一定能兼容新数据库结构。数据库恢复可能覆盖或丢失新数据，应先在隔离环境演练、备份当前数据库，再按经过验证的恢复方案处理，不要直接在生产库执行未经测试的恢复命令。

```bash
sudo systemctl status myapp
sudo journalctl -u myapp -n 100 --no-pager
sudo nginx -t
sudo systemctl status nginx
```

## 10. 参考资料

- [Ubuntu Server：UFW 防火墙](https://ubuntu.com/server/docs/firewalls/)
- [Go 官方下载页](https://go.dev/dl/)
- [Go 官方安装说明](https://go.dev/doc/install)
- [go-sql-driver/mysql：DSN 说明](https://github.com/go-sql-driver/mysql#dsn-data-source-name)
- [Certbot 使用文档](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [MySQL 8.0：mysqldump 备份与恢复](https://dev.mysql.com/doc/refman/8.0/en/using-mysqldump.html)
- [systemd.exec：服务安全限制](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)
