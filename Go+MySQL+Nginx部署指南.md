# Go + MySQL + Nginx 网站部署指南

> 本文记录在全新 Ubuntu 云服务器上部署 Go 后端网站的流程：系统初始化、安装 Nginx 和 MySQL、安装 Go、上传并编译项目、配置 Systemd、配置 Nginx 反向代理，以及绑定域名和启用 HTTPS。
>
> 本文是独立的 Go 部署记录，不替代仓库中原有的 WordPress + 宝塔面板搭建记录。

---

## 一、部署环境与目录约定

本文示例使用以下环境。实际操作时，请将示例 IP、域名、项目目录和数据库信息替换为自己的配置。

| 项目 | 示例值 |
| :--- | :--- |
| 操作系统 | Ubuntu 22.04 LTS |
| 后端语言 | Go（示例版本 1.23.4） |
| 数据库 | MySQL 8.0 |
| Web 服务器 | Nginx |
| 项目目录 | `/opt/myapp` |
| Go 服务监听地址 | `127.0.0.1:8080` |
| 示例服务器 IP | `8.133.191.82` |
| 示例域名 | `zyyjy.top` |
| 前端静态文件目录 | `/var/www/myapp` |

**部署结构：**

```text
浏览器
  |
  | HTTP / HTTPS
  v
Nginx（80 / 443）
  |                         |
  | 静态页面请求             | /api/ 请求
  v                         v
/var/www/myapp          Go 后端（127.0.0.1:8080）
                              |
                              v
                         MySQL（本机）
```

---

## 二、步骤 0：系统初始化与基础工具安装

拿到全新的 Ubuntu 云服务器后，先更新软件源并安装常用工具。

```bash
# 更新软件源和已安装的软件包
sudo apt update && sudo apt upgrade -y

# 安装常用工具
sudo apt install -y curl wget git unzip vim ufw

# 放行 SSH、HTTP、HTTPS 端口
# 注意：启用防火墙前，务必先放行 SSH（22），避免失去远程连接
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443

# 启用防火墙
sudo ufw enable

# 查看防火墙状态
sudo ufw status
```

同时检查云服务器厂商控制台中的安全组规则，确保也放行 TCP 22、80、443 端口。若应用只通过 Nginx 对外提供服务，通常不需要将 Go 的 8080 端口或 MySQL 的 3306 端口开放到公网。

---

## 三、步骤 1：安装 Web 服务器（Nginx）

Ubuntu 官方软件源提供 Nginx，可以使用 apt 安装。

```bash
# 安装 Nginx
sudo apt install -y nginx

# 设置开机自启并立即启动
sudo systemctl enable --now nginx

# 检查运行状态
sudo systemctl status nginx
```

**验证方式：** 在浏览器访问服务器公网 IP。如果看到 Nginx 欢迎页面，说明 Nginx 已成功启动。

---

## 四、步骤 2：安装关系型数据库（MySQL）

使用 apt 安装 MySQL 服务端。

```bash
# 安装 MySQL
sudo apt install -y mysql-server

# 设置开机自启并立即启动
sudo systemctl enable --now mysql

# 检查运行状态
sudo systemctl status mysql
```

### 2.1 创建项目数据库和专用用户

进入 MySQL：

```bash
sudo mysql
```

在 MySQL 命令行中执行以下 SQL。请将示例密码替换为自己生成的强密码，不要把真实密码提交到 GitHub：

```sql
CREATE DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'myapp'@'localhost'
  IDENTIFIED BY '请替换为强密码';

GRANT ALL PRIVILEGES ON myapp.* TO 'myapp'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

这里创建了独立的 `myapp` 数据库和本机专用数据库用户，Go 程序不应使用 MySQL 的 root 账号连接业务数据库。

---

## 五、步骤 3：安装 Go 语言环境

Ubuntu 软件源中的 Go 版本可能不是项目需要的版本。以下按 Go 官方 Linux amd64 压缩包安装示例版本 1.23.4。若服务器是 ARM 架构，请选择对应的 ARM64 安装包。

```bash
# 下载 Go 安装包（示例版本 1.23.4）
cd /tmp
wget https://go.dev/dl/go1.23.4.linux-amd64.tar.gz

# 如果已有旧版 Go，先移除旧安装目录
sudo rm -rf /usr/local/go

# 解压到 /usr/local
sudo tar -C /usr/local -xzf go1.23.4.linux-amd64.tar.gz

# 创建命令软链接
sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go
sudo ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt

# 验证安装
go version
```

配置 Go 模块代理。示例使用国内 Go 模块代理；如果服务器网络环境不需要代理，也可以使用 Go 默认设置。

```bash
go env -w GOPROXY=https://goproxy.cn,direct
go env GOPROXY
```

> Go 版本应以项目的 `go.mod` 文件要求为准。本文的 1.23.4 是部署示例版本，不代表所有项目都必须使用该版本。

---

## 六、步骤 4：将 Go 项目代码上传到服务器

假设服务器 IP 为 `8.133.191.82`，项目部署目录为 `/opt/myapp`。

先在服务器创建目录：

```bash
sudo mkdir -p /opt/myapp
```

有两种常见上传方式，选择其中一种即可。

### 方式一：使用本地 scp 命令上传

在自己的电脑终端（例如 Windows PowerShell）中执行。将本地项目路径替换为真实路径：

```powershell
scp -r "你的本地Go项目目录\\*" root@8.133.191.82:/opt/myapp/
```

如果使用 Windows PowerShell，通配符和目录复制行为可能受本地环境影响；也可以先压缩项目，再上传压缩包并在服务器解压。

### 方式二：通过 Git 拉取

先将本地代码推送到 GitHub 或 Gitee，然后在服务器执行：

```bash
cd /opt
sudo git clone https://github.com/你的用户名/你的仓库名.git myapp
cd /opt/myapp
```

如果仓库是私有仓库，需要先配置安全的 Git 身份验证方式。不要把访问令牌直接写进公开命令或文档。

---

## 七、步骤 5：编译 Go 后端

在服务器进入项目目录，整理依赖并编译 Linux 可执行文件。

```bash
cd /opt/myapp

# 整理并下载项目依赖
go mod tidy

# 编译为 Linux 可执行文件
go build -o myapp .

# 添加可执行权限
sudo chmod +x myapp
```

如果项目使用 CGO 或依赖特定系统库，编译环境还需要满足项目自身的要求。编译前请确认项目入口包位置；如果入口不在项目根目录，应将最后一条编译命令中的 `.` 改为实际入口目录。

### 5.1 配置文件和数据库连接

Go 项目读取数据库地址、用户名和密码的方式取决于项目代码。常见做法是通过环境变量或配置文件提供连接信息。请根据项目实际使用的变量名配置，切勿将真实数据库密码写入公开仓库。

如果项目使用环境变量，可以将生产环境配置放在服务器本地的环境文件中，并限制读取权限。例如：

```bash
sudo mkdir -p /etc/myapp
sudo nano /etc/myapp/myapp.env
```

文件内容示例（变量名仅作示意，必须与项目代码保持一致）：

```ini
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=myapp
DB_USER=myapp
DB_PASSWORD=请填写实际数据库密码
```

限制文件权限：

```bash
sudo chmod 600 /etc/myapp/myapp.env
sudo chown root:root /etc/myapp/myapp.env
```

---

## 八、步骤 6：配置 Systemd 服务

使用 Systemd 可以让 Go 程序在后台稳定运行，并支持开机自启和异常退出后自动重启。

创建服务文件：

```bash
sudo nano /etc/systemd/system/myapp.service
```

填入以下内容：

```ini
[Unit]
Description=Go Web App
After=network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/opt/myapp
EnvironmentFile=-/etc/myapp/myapp.env
ExecStart=/opt/myapp/myapp
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

确保服务用户能够读取程序和项目所需文件：

```bash
sudo chown -R www-data:www-data /opt/myapp
sudo chmod +x /opt/myapp/myapp
```

如果项目需要写入特定目录，应只对对应目录授予写权限，不要无差别开放整个系统目录的权限。

保存后执行：

```bash
# 重新加载 Systemd 配置
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable myapp

# 启动服务
sudo systemctl start myapp

# 查看运行状态
sudo systemctl status myapp
```

查看程序日志：

```bash
sudo journalctl -u myapp -n 100 --no-pager
sudo journalctl -u myapp -f
```

> 只有当程序实际监听在 `127.0.0.1:8080` 时，后续 Nginx 配置才能按本文示例工作。请检查 Go 项目的监听地址和端口。

---

## 九、步骤 7：配置 Nginx 反向代理

前端静态文件由 Nginx 直接返回，后端 API 请求转发给本机 Go 服务。

创建 Nginx 站点配置：

```bash
sudo nano /etc/nginx/sites-available/myapp
```

填入配置。将域名替换成自己的真实域名：

```nginx
server {
    listen 80;
    server_name zyyjy.top www.zyyjy.top;

    # 前端静态资源目录
    root /var/www/myapp;
    index index.html;

    # 前端路由回退
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 将 API 请求转发给 Go 后端
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

创建前端目录（如果项目有前端页面）：

```bash
sudo mkdir -p /var/www/myapp
```

将前端构建产物中的 `index.html` 和静态资源放入该目录。如果项目没有独立前端页面，可以按项目实际情况调整或删除静态文件相关配置。

启用站点并检查配置：

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp

# 如果默认站点占用了默认配置，可按需禁用
sudo rm -f /etc/nginx/sites-enabled/default

# 检查 Nginx 配置语法
sudo nginx -t

# 平滑重载配置
sudo systemctl reload nginx
```

---

## 十、步骤 8：绑定域名并配置 HTTPS

### 8.1 配置 DNS 解析

在域名服务商的 DNS 管理页面添加记录：

| 类型 | 主机记录 | 记录值 |
| :--- | :--- | :--- |
| A | @ | 服务器公网 IP |
| A | www | 服务器公网 IP |

等待 DNS 生效。可以在本地或服务器使用以下命令检查解析结果：

```bash
nslookup zyyjy.top
```

确认域名解析到当前服务器公网 IP，并且云服务器安全组和 UFW 已放行 TCP 80、443 端口。

### 8.2 使用 Certbot 申请 SSL 证书

安装 Certbot 和 Nginx 插件：

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

为域名申请并自动配置证书：

```bash
sudo certbot --nginx -d zyyjy.top -d www.zyyjy.top
```

根据提示填写邮箱并同意服务条款。若希望 HTTP 自动跳转 HTTPS，可在 Certbot 提示时选择重定向选项。

测试证书自动续期：

```bash
sudo certbot renew --dry-run
```

---

## 十一、常用运维命令

| 目的 | 命令 |
| :--- | :--- |
| 查看 Go 版本 | `go version` |
| 查看 Go 代理 | `go env GOPROXY` |
| 查看 MySQL 状态 | `sudo systemctl status mysql` |
| 重启 Go 服务 | `sudo systemctl restart myapp` |
| 查看 Go 服务状态 | `sudo systemctl status myapp` |
| 查看 Go 服务日志 | `sudo journalctl -u myapp -f` |
| 重启 Nginx | `sudo systemctl restart nginx` |
| 检查 Nginx 配置 | `sudo nginx -t` |
| 重载 Nginx 配置 | `sudo systemctl reload nginx` |
| 查看端口监听 | `sudo ss -tlnp` |
| 查看内存使用 | `free -h` |
| 查看进程资源 | `top` |
| 查看防火墙状态 | `sudo ufw status` |

---

## 十二、部署完成后的检查清单

- [ ] 云服务器安全组已放行 22、80、443 端口。
- [ ] UFW 已放行 SSH、HTTP、HTTPS，且可以继续通过 SSH 登录。
- [ ] Nginx 服务正常运行，访问域名可以返回页面。
- [ ] MySQL 服务正常运行，项目数据库和专用用户已创建。
- [ ] Go 版本满足项目 `go.mod` 的要求。
- [ ] Go 项目已成功编译，数据库连接配置与代码一致。
- [ ] Systemd 中 `myapp` 服务状态为 `active (running)`。
- [ ] Go 服务只在预期地址和端口监听。
- [ ] Nginx 的 `/api/` 请求能够正确转发到 Go 后端。
- [ ] DNS 已解析到服务器公网 IP。
- [ ] HTTPS 证书申请成功，自动续期测试通过。

---

## 十三、部署过程中需要注意的事项

1. **不要公开真实密码：** 数据库密码、SSH 密码、访问令牌等应保存在服务器本地的安全配置中，不要提交到 GitHub。
2. **不要随意开放数据库端口：** 如果 Go 和 MySQL 在同一台服务器上，优先让 Go 通过本机连接 MySQL，不要将 3306 暴露到公网。
3. **确认端口一致：** Go 程序实际监听的端口必须与 Nginx 的 `proxy_pass` 地址一致。
4. **确认目录权限：** Systemd 的运行用户必须能够读取可执行文件、配置文件和项目运行所需资源。
5. **先检查再重载：** 修改 Nginx 配置后，先执行 `sudo nginx -t`，确认无误再 reload。
6. **按项目实际情况调整：** 本文中的数据库环境变量名称、Go 入口目录、前端目录和 API 路径均为示例，需要与自己的项目代码匹配。

---

## 十四、实践总结

通过这套部署流程，可以完整实践 Go 网站从代码上传到公网访问的主要环节：

- Linux 服务器初始化与防火墙配置
- Nginx Web 服务安装与管理
- MySQL 数据库创建及专用用户授权
- Go 运行环境安装与模块代理配置
- Go 项目上传、依赖整理和 Linux 编译
- Systemd 后台服务、开机自启与日志排查
- Nginx 静态资源托管与 API 反向代理
- DNS 域名解析和 Certbot HTTPS 证书配置

> 本文记录的是通用部署流程。真正部署成功还取决于 Go 项目的代码结构、数据库连接实现、前端构建方式和实际监听配置。
